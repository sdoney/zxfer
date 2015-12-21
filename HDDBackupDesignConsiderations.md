# Design considerations for HDD-based backups using ZFS/zxfer #

Now that I have hopefully [established](HDDBackupsWithZxfer.md) that using HDDs in a ZFS system and transferred using the zxfer script can be a sensible backup solution, I will walk through some design decisions I have made in order to come up with a practical implementation. Hopefully this will serve as a guide to others to make their decisions easier, and/or to provoke comment and lead to a better overall solution. Note that this is only a discussion and I will NOT be held responsible for any loss of any kind you have as a result of reading this article.


## Hidden costs of poor HDD reliability ##

It is readily evident to anyone who has had to replace several HDDs that there are numerous costs that are incurred with drive failures. Some examples:

  * Cost of shipping the drive back to RMA. (Often not covered by the manufacturer.) Or if you judge that you cannot return the drive for security reasons, the cost of the drive itself and cost of destruction, if you go that route.
  * Cost of your time:
    * In testing the drive in accordance with the manufacturer's testing suite to get an error code for them, if relevant.
    * In writing the drive with random data several times in order to render the drive unreadable to the manufacturer. If you don't want to risk sending private or commercially sensitive information back to the manufacturer, this is what must be done (at a minimum).
  * In wrapping the drive appropriately in accordance with the manufacturer's specifications to be sent back, and posting it.
  * The cost of buying spare drives to cover downtime, or if you don't buy spare drives, the risk of a disaster happening during this time.
  * The cost of the lowered lifetime of a drive after RMA period is up. e.g. if a drive has a 3 year warranty and you want to use it 5 years, a more reliable drive has greater likelihood of not dying until the end of desired use.
  * The costs of extra redundancy required to counteract the increased risks of lost data through bad sectors or outright failure. Remember that even though a triple mirror or RAIDZ2 protects us against 2 disks failing, it won't protect us against 2 disks failing and some bad sectors on the third, or 1 disk failing and bad sectors in the same places on 2 other disks. This redundancy can come about with increased copies on ZFS, higher levels of RAID (e.g. RAIDZ3) or mirror, and perhaps more frequent backups, but any way you slice it there is a cost.

## Google's [Failure Trends in a Large Disk Drive Population](http://static.googleusercontent.com/external_content/untrusted_dlcp/labs.google.com/en//papers/disk_failures.pdf) - Implications ##
  * HDD failure rates highly correlated with model/manufacturer/vintage → i.e. research and buy known good drives
  * Minimal correlation of failure with activity/temperature. 37-45% is ideal range according to their results.
  * Certain SMART data (e.g. scan errors, reallocation errors, offline reallocations and probational counts) indicates 10x risk of failing → Monitor and replace as soon as detected. **Critical threshold for scan errors is 1.**
  * Some drives still fail without precursors from SMART.
  * Results are based on drives of up to 400GB of size only. Comparing such drives to modern > 1TB drives through review sites and through my own personal experience, I think it is safe to say that modern >1TB HDDs are less reliable. Even back then, average annual failure rates were over 8% for a 3 year old drive.

From this, my conclusion is that if we want reliability for the minimum cost, we need to first find those models of HDD that are the most reliable, subject them to an initial burn-in test (much like Google does), and then monitor them for SMART data. As soon as we get an error in (scan errors, reallocation errors, offline reallocations and probational counts), we RMA the HDD if possible.


## Finding reliable HDDs ##

In the absence of any better methodology, I use sites with statistically relevant numbers of reviews. In practice this means newegg, then as a distant second, amazon.

We want to find the percentage of drives that are returned because of failure. To do that, we have to look at the reviews. As a sad testament to the unreliability of the modern large HDD (e.g. 1TB and above), you will find out that people are willing to award as many as 3 stars for a HDD that fails on them (providing that they have no issues with RMA). So we can use the combined 3 stars and below percentage to get an idea of the reliability of a particular drive, with some provisos. See [#Newegg\_HDD\_reliability\_metric](#Newegg_HDD_reliability_metric.md)

HDD failures happen anywhere within the 5 or so years that HDDs are typically used for. Unless there is an excessively high DOA rate, that means that reliability will not have much of an impact on a drive's initial reviews. At first, factors not relevant to someone who desires to use the HDDs as backup will predominate: speed, noise, power usage and cost/GB are some of the factors. These are indeed nice to have but pale in comparison to reliability. And in a market where reliability is rarely found, we must ignore those niceties until the manufacturers figure out how to make reliable multi-TB drives.

In this early stage of a drive's life, the <=3 star review percentage is only good for showing a lower bound on the failure likelihood of the drive. e.g. A lemon will have a high <=3 star review percentage, if for some reason it is prone to many DOAs and early failures. However, other lemons yet to show their true colors will have low <=3 star review percentage. So at this stage it is only useful to tell you which drive NOT to take a chance on.

After a while, drive failures start to happen in accordance with the reliability of the drive model. What then happens is that irate owners go back to “neg" the particular model that they have bought. While this may bias those ratings negatively (as content owners are probably less likely to leave a review), I can think of no reason why a Seagate buyer would be more likely to do this than a Samsung buyer. So as long as we are using this to compare different drive models and not estimating failure percentages of drives we buy, <=3 star review percentage is probably the best methodology available to the average consumer. The longer the drive has been in the market, the better indication of true drive failure rate it will be.


### Other HDD considerations ###

It is always a good idea to do a quick google before you buy HDDs to find out if there have been any issues, particularly with the way HDDs can [lie](http://forums.freebsd.org/showthread.php?t=17036|http://forums.freebsd.org/showthread.php?t=17036) to the system about the block sizes.


## System interface choices: e-SATA vs USB3 ##

### SMART ###

E-sata wins here, as the SATA interface appears necessary to read the SMART data (which is useful to check if a HDD is failing). Only if you are willing to forsake SMART data should you go with an exclusively USB3 solution. Given how easy it is to check SMART data, and how usefully SMART data predicts a failing drive, it makes a lot of sense to go with the eSATA solution.



USB3 does have the handy ability to use cheap multi-port hubs, which are convenient if you are backing up to some sort of RAIDZ, RAIDZ2 or RAIDZ3. However, you may be able to port multiply your SATA ports using just 1 e-SATA port, if your controller supports this. Or if you have lots of internal SATA ports to spare, just buy some extra “back plates" or 5.25" front panels with eSATA - ebay is your friend! Or just go with a metre long SATA to e-SATA cable and have it exit your case somewhere.


### Connecting your SATA drives via e-SATA ###

There are two main methods: HDD docks and enclosures. HDD docks are cheaper and able to easily adapt to the number of drives in your pool, as you can use the same HDD dock(s) for however many backup pools you have. There are also HDD docks that suit multiple (as many as 4 I have seen) drives. I am a bit wary of the vibration issue with multiple drives in the same unit, but it may be no worse than 4 drives connected to a typical computer case via screws, or mounted next to a vibrating optical drive. Also, toting 6 drives in a bag is less heavy than 6 drives in a steel drive enclosure. Plugging in and connecting the pool would be quicker with the enclosure.


### Speed ###

You will want to make sure that you aren't saturating individual SATA ports with multiple drives.  e.g. If you have a drive with 150MB/s average write speed, you don't want to connect more than two of them to a single SATA2 e-SATA port. In that case you would stick with providing an e-SATA port for every 2 HDDs, and buy dual HDD docks. Of course, if you had SATA3 ports, cables, docks and HDDs, you could get away with a single quad HDD dock. I have not yet researched what that requires as SATA2 does the job for now.


### Other implications – the more SATA ports, the better, and port multiplication ###

Note that if you are in the market for a motherboard, you will note that it makes good sense to look for as many SATA ports as you can find. e.g. a root mirror SSD, plus a L2ARC/ZIL SSD, plus a triple mirror storage pool, plus an optical drive is already 7 SATA ports. You will need at least one more SATA/e-SATA port for a single mirror backup alone (with good speed). For every 2 drives in your backup pool, add an extra SATA port.

Port multiplication support is a must, unless you want to use single HDD docks and a SATA port for every HDD dock connected.


### HDD physical protection ###

You can somewhat mitigate the risk of drive damage in the e-SATA HDD dock solution by purchasing silicone drive covers for each drive. These greatly minimize any jarring when the drives are placed down on hard surfaces, and also stop drives sliding around (and off) the table. They also enable easy and stable stacking of the drives since they interlock.

Another thing you might consider if you use the silicone drive covers is that it is probably easier to get a SATA data + power extension for each port on your HDD dock. That way you can sit your silicone encased HDDs down without having to remove the silicone case every time - there is a convenient hole for this purpose. Heat may be an issue but for backups of short duration it will be OK. After an extended period of use I found the temperature differential was 25 degrees Celsius above ambient (according to `# smartctl -a`). If you are worried about this a cheap portable fan directed at the drives would probably provide enough cooling.

For transporting HDD from one location to another, you will want to carry them in some sort of padded container. Camera bags work well for this purpose (e.g. Lowepro) ~ $60 on ebay. Even a cardboard box lined with bubble wrap in a cloth bag will work. At least one location should house the drives in a fire/water proof safe.

## Mirror, RAIDZ2, RAIDZ, copies, prayer ##

There is quite an array of choices we can make here for our backup virtual devices, depending on convenience, cost and how redundant we want to make them. The main concerns as I see them:

  * how reliable your backups should be
  * what sort of spare, unused HDDs you have around that can be used, and
  * how much you value the data that you store vs the size of that data (e.g. media library, documents, family photos, mission critical business database, etc). If you have multiple categories of data, it probably behooves you to adopt a different strategy for each one.
  * how much you are willing to risk losing a day or two of data if you have other backups that hold slightly more dated info.
  * whether you want to [complicate](http://www.miracleas.com/BAARF/BAARF2.html|http://www.miracleas.com/BAARF/BAARF2.html) [things](http://constantin.glez.de/blog/2010/01/home-server-raid-greed-and-why-mirroring-still-best|http://constantin.glez.de/blog/2010/01/home-server-raid-greed-and-why-mirroring-still-best) by moving to RAIDZ2, or just use mirrors. Mirrors are better, though cost of equipment is higher.

It seems that in mid 2011, 2TB drives are at the sweet spot of cost/GB, with 3TB only being about 15% more expensive per GB. 1 or 1.5 TB are not much more expensive though per GB. As long as we only choose reliable drives, we can pick and choose between the 1-3TB range based on what is convenient and reliable, without that much thought for cost.



Remember also that if we scrub the backup pool and check SMART status after each backup in order to catch failing disks early, a backup only has to last from off-site storage to be replicated to your new system. There is much less chance of a physical failure striking during a restore, compared to sitting a year in a server. Since a restore probably lasts less time than a day, there is probably 365 times less chance of a disk failing, though it is of course higher being physically moved around.



So I think in some cases provided that we use multiple off-site backup virtual devices in a rotation, we can use a single drive for each backup virtual device. It would certainly be an improvement on the practices of many people. I would recommend using copies=2 or 3 in order to guard against bad sectors. For more important information, mirrors are a better choice. If we don't use copies on a mirror, we get protection against drive failure or corruption on one disk. If we use copies=2 on a mirror, we get protection against both a drive failure and corruption on the remaining disk. We can mitigate the expense somewhat by using compression if the data is suitable. Unfortunately, most media is already highly compressed so we gain nothing by compressing it again.



I would restrict the use of RAIDZ, RAIDZ2 and RAIDZ3 to the cases where we are dealing with large quantities of data, and are greedy. The reason for that is that it's handy to be able to buy larger drives as they will be a  useful size for longer, unless the larger drives are a natural fit for a RAIDZ/2/3 of your data. With RAIDZ, rather than using copies to provide the redundancy we use redundant disks. e.g. If you have a 6 disk RAIDZ2, 2 disks are redundant meaning that 2 drives failing or corrupting are covered for only an extra 6/4 -1 = 50% in cost. It may be more reliable than a mirror or single disk with copies, and cheaper. However, do not that re-silvering times can be very lengthy.



The tradeoff with RAIDZ vs RAIDZ2 or RAIDZ3 is yours to make. However, if one disk in your RAIDZ dies, you have very little margin to work with. So I think it would be better to drop down a size disk and use RAIDZ2 if cost is a concern rather than go with RAIDZ. Do realize that the more disks you use, the more HDD docks, spare SATA ports and e-SATA connections you will have to have.


### Categories versus backup pool vdev configuration ###

**Media library**: mirror, RAIDZ2 or single disk. Since most media is highly compressed, there is next to no gain for compressing, and hence compression + copies=2 is not a very effective option. It is better to mirror for redundancy rather than use copies. Assuming you have original DVDs or CDs, you might chance it with single drives.

**Family photos and documents**: mirror. However, 3 or 4 multiple single drives with copies=2 is probably not terrible, since if you inadvertently destroy a backup, you will still have most of your stuff and can afford to not sweat too much with your next restore attempt.

**Mission critical data**: triple mirror, with multiple backup pools. If cost is a factor, I think it would be better to drop down to mirrors or even single disk/copies=2 and have more backup pools than have more redundancy in each pool itself. I suspect that there is more chance of a tragic mistake being made with the heightened stress that comes with a disaster recovery scenario than two drives dying at once.


## Number of backup pools ##

This was covered in the previous article. To recap - I would suggest minimum 3 backup pools in daily rotation. More is obviously better. Several more taken at infrequent intervals will guard against the backups being inadvertently destroyed or corrupted through various means.


## Storage locations ##

You want to have as many storage locations as possible, provided that they are trustworthy. To the extent they are not trustworthy, the backup pools or at least the drives themselves (e.g. in FreeBSD, with [geli](http://www.freebsd.org/doc/handbook/disks-encrypting.html) - see [#Encryption](#Encryption.md)) must be encrypted.



You need to take Murphy's Law into account with selecting your backup locations. Anything that can go wrong, will go wrong and at the worst possible time. At least one of every backup location should be immune to likely disasters in your area, e.g. not limited to fire, theft or destruction (including by employee), flood, tornado, hurricane, tsunami, earthquake, terrorist attack, etc. For some of these there is no choice but to have several backup locations.


### What to do with failed drives ###

It is highly likely that some drives will die. If you want to be able to RMA those drives, you should also consider encrypting any drives before you store important data on them. There is a more detailed [discussion](HDDBackupsWithZxfer.md) about this in the previous article. Be aware that encryption can be a serious foot gun. If you plan on encrypting anything important, losing the key is a disaster in and of itself. If you lose the key and all your files and backups are encrypted with the same key, you have just lost all your data, permanently. DO NOT EVER LOSE THE KEY. Key protection is at least as important as backup.


## Conclusion ##

Hopefully this should give a good overview of the equipment and setup needed to begin backing up with ZFS and zxfer. If you have better solutions, feel free to suggest them.


---

## Footnotes ##

### Newegg HDD reliability metric ###
See this [post](http://forums.storagereview.com/index.php/topic/28230-most-reliable-hard-drive/page__st__20|http://forums.storagereview.com/index.php/topic/28230-most-reliable-hard-drive/page__st__20) - FWIW I came to the same conclusion independently, but after googling I see that at least one other person on the web gotten there first. Kudos to them. As someone on that forum says 'In on-line review research, the first rule is "read all the words"'. I do exactly that, and came to exactly the same conclusions. Three cheers for OCD.


### Encryption ###
As of May 2011, zxfer does not support transferring encrypted ZFS file systems. It should work with geli though.
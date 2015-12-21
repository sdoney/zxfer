# HDD based backups with ZFS & zxfer – a new paradigm #

As I was writing the man page for [zxfer](http://www.zxfer.org) and reading some user comment, it became evident that it wasn't obvious to everyone how zxfer should be used. I also realized that the man page really wasn't the best place to compare and contrast backup strategies at length, hence the need for this article.



I will review the use of tape backup systems and touch on the important reasons why they have been popular. I will then outline how a system of backing up to HDD based ZFS pools using zxfer to transfer from primary storage pool to backup pool can work, and how it can fulfill most if not all of those same needs. After that I will address some setup considerations. This article is for educational purposes only, I will not be held responsible in any way for anything you do as a result of reading this article.



This information is based on my own personal workstation backup and restore procedures, which I have tested and appear to work reliably enough for my purposes.  At present my backups consist of several offline HDD pools that are ZFS mirrors. The writing of this article also helped me to think about some things I had not questioned before, and I will duly incorporate any improvements into my own systems.




Do note that in the process I do not intend to disparage tape. Tape certainly has its place, especially in institutions. It would be a shame if we lost valuable historical data just because an organization went unfunded for a few years and there was no money to buy HDDs. I believe that humanity should have archival quality media that is capable of recording the history of our civilization in much the same way that paper has throughout the years. I'm not sure whether tape is that media, but it is without doubt a few steps closer than HDDs in that direction.


## Tape backup systems ##

Typically, backup data is written incrementally to tapes that are rotated in some systematic fashion and stored off-site. One of the more thorough backup rotation schemes is [Grandfather-Father-Son](http://en.wikipedia.org/wiki/Backup_rotation_scheme) (GFS). This typically involves writing a backup to a set of 7 tapes over the course of a week. These are the “sons". At the end of the week, one of the tapes is kept as a representative of that week (a father). At the beginning of the next week, the oldest son from the previous week has incremental backup information written to it. And so on and so forth just as in the First In First Out (FIFO) system.



After a month, there will be 4 weeks of “fathers" that have built up. One of those fathers is kept to become a “grandfather", or representative of that month. Meanwhile the other weeklies are recycled just like the dailies are. One or more of the grandfathers is often kept as a representative of the year or quarter, and henceforth kept forever. LTO tape is reputed to last 25+ years, so far in the future these grandfathers should be transferred to some newer media that can be read by the equipment of the day.



Using a GFS tape backup setup, we have the following advantages/features:

  * the possibility to recover data that has been destroyed either intentionally or unintentionally, going back as long as the organization exists. e.g. If someone in your organization deletes an email that will prove something in a court case that is worth $$$, you can recover it.
  * a resilient archival format that lasts 20+ years, with built-in error correction.
  * due to tape resiliency, we can effectively ignore issues of RMA and security. Any bad tapes you can keep on hand without much cost, as there should be few of them.
  * some protection against data corruption in your main storage array, again, going back years.
  * protection against disk failure.
  * mitigation of data loss in the case that your system gets destroyed or is compromised - i.e. capable of off-site, offline storage. Data should be able to be restored in full from some date before the date the system was compromised.
  * resources geared towards protecting data that is most likely wanting to be recovered, e.g. someone deletes a file and they want the newest version within a week.
  * physically quite robust, probably not very fireproof but very shock resistant.
  * speed in writing the backup, as only the increment is transferred.
  * historical snapshots of data usually aren't destroyed by a single mistake.



We also have the following disadvantages:

  * Recovery is tedious. While the speed at which data can be written to tape is quite fast, random access is very slow and inconvenient. It's very much like trying to get to your favorite part of a movie on a VHS tape - it takes time.
  * Initial cost is high and not suited to organic growth.
  * You may not be transferring snapshots to the tape. In this case, GFS as outlined above will require you to keep about 7+4+12 = (23 + N) x the size of your data set, where N is the number of years you have kept.

Also note that unless you have a few off-site locations that take into account local or man-made disaster probabilities, your backups might not be as safe as you think. If you have all your tapes stored at the one location and it burns down or gets hit by a tsunami at the same time as your server dies, you are no better off than with 1 HDD pool stored at one location. Or if a sysadmin has access to every backup location and becomes disgruntled, the organization may also lose all its data. That is not to say that the HDD pool is superior, just that in practice there may be no difference between the two in terms of data security.


## Mapping tape backup concepts to HDD based backups using ZFS/zxfer ##

### Operating System ###

I will explain how the new system will work, and draw parallels with tape backups where I can. First of all, you have to use an operating system with ZFS. So in practice that means FreeBSD 8+ or Solaris.




### Snapshots ###

On your main storage pool, you will run a snapshot management tool via a cron job that automatically creates and deletes snapshots so that over time you are left with a GFS scheme of snapshots (e.g. dailies, weeklies, monthlies, quarterlies or yearlies). (See [#GFS](#GFS.md)) This is easily possible because there is no limit to the number of snapshots that can be created, and only the differences between snapshots are stored. This means that in practice for many patterns of data accumulation there is little disadvantage to the GFS system. Also consider how storage space per dollar and per HDD has increased exponentially and will probably continue to do so for some time. Even for data from a few years ago that has vastly changed (and hence has a snapshot near equivalent in size to the total size of data at the time), there is not that much of it compared to current storage capacity.



Having this GFS scheme of snapshots on your main machine means that in most cases restoration is as simple as delving back into the .zfs/snapshot directory and copying across the required file. No tape loading required unless the main server is destroyed! This practice alone makes most restoration much more practical and less tiresome for the admin.



The big conceptual difference between tape and ZFS HDD pool based backup is the divorce from the snapshot and the media. At each backup event, the snapshots on the main system are synced with the snapshots on the backup. In this way, we keep a long historical record in both primary system and every backup pool without being reliant on the longevity of any individual drive.


### Backup pools ###

For actual off-site, offline backups, several backup pools are necessary. Note that these pools are only online when they are connected to the server, other than that they are turned off and not connected to a power supply. Keeping them plugged in makes them both expensive to run in terms of electricity, and brings increases risk due to power surges.



I would suggest at minimum 3 such pools and preferably 5-7. Each should have some redundancy. If it is important data at least 2 disks should be able to fail without causing the pool to be unreadable. For small amounts of less important data, ordinary mirrors or single HDDs can be used, but increasing the copies kept on the ZFS filesystem is strongly recommended.



Note that in the same way as tapes have compression, you can use zxfer to temporarily override compression (and in future, deduplication) settings on the backup, and later painlessly restore the original settings when you restore. This allows a measure of cost savings for data that is not already highly compressed (e.g. most media).



There should be at least 3 pools used in regular daily rotation because if the server dies during a backup operation and takes that backup pool with it, we should not be dependent on the one last backup for recent files. Having another two backup pools in service will allow backups to be taken at infrequent intervals, for example every month using one and then the other. For more paranoid people, the same thing can be done every year or so with another backup pool. This practice will give a year for a system compromise to be found so that data can be compared with a known-good backup to determine what has possibly been changed and what hasn't. If these backups are stored by someone other than the sysadmin, they also allow a degree of mitigation of the risk of a rogue sysadmin (which is probably overstated - see [#Rogue\_Sysadmin](#Rogue_Sysadmin.md) ) .



It might be still advisable to keep more backup pools to give a few years of protection just in case, but it is questionable how many organizations keep their backup tapes out of the reach of a single sysadmin anyway, since tapes could be re-written in much the same way. It wouldn't be trivial to doctor a backup pool to have changed snapshots, as snapshots are read-only. It could be done, but it would probably be a similar amount of hassle on either tape or HDD based backups.


### Backup pool interfaces ###

Moving on to the method of connecting backup media to the server - instead of tape drives, we would use e-SATA HDD docks. Dual HDD docks are probably the most practical as they should not yet saturate a single e-SATA connection. Since an e-SATA connection is achieved by simply connecting an internal SATA port with an e-SATA plug by means of a cable, it is not difficult to equip with e-SATA a machine that has been suitably provisioned with spare SATA ports. A very cheap solution is just to pass a SATA to e-SATA cable through the case, as they can be up to 1m long.



I recommend labeling the power supplies for each type of HDD dock you buy. Despite having the same connector on the power supply/HDD dock interface, many are not compatible. I have personally destroyed two HDDs by disregarding this, so in the event of a restore it would be worthwhile to double check that your HDD docks match their power supplies, and even test an old HDD before inserting your valuable restore drives.



In order to keep the HDDs from being damaged in transit, silicone sleeves for the individual HDDs can be used. They can then be packed into a shock resistant camera bag or similar. Note that the HDDs are only online and on-site when the backup (and HDD health checks, e.g. SMART status and # zpool scrub) is happening. While it is possible to backup to an online system, it should never be the last ditch line of defense. It is probably a good idea to also buy extension SATA and power cables that go from the HDDs to the HDD docks. You can then leave the HDDs in their cases. If worried about heat, a cheap fan blowing on them may provide adequate cooling.




### What to do with failed drives ###

We have to plan for one other issue with this system, and that is what should be done with failed drives. Drives can be counted on to fail, and if we are to use our manufacturer's warranty, we have to ask some questions. e.g. Do we trust the manufacturer to not read our data? We have no real way of knowing that this is not taking place, and it would be potentially an excellent method for state sponsored industrial espionage. Consider that many drives fail with no damage to the platters, and no chance to perform even a minimally secure delete. Do companies with some juicy data just send these off to “save" a minimal $100? I bet they do. Can many of these drives be read by just changing out a faulty circuit board? I assume so. Does someone on the other end read the data? I'm going to assume that they do. No company is going to volunteer that they read your HDD data, whether this practice actually takes place or not.



Once some success and profit may be had with minimally failed drives, it would only be natural to fund such an effort. That funding would lead to increasingly sophisticated data recovery. Also all the technical expertise of a hard drive manufacturer would be a tremendous help. While it is unknown the extent to which technology exists to break encryption or read overwritten drives, several things are certain:

  * Those who have invested in developing such technology are highly unlikely to publicly declare their capabilities.
  * The only way to be absolutely certain that someone can never read data from a HDD is to either never write to it, or if it is written to, randomly mix the platters completely on a finer level than the magnetic domains themselves. The most sure method of HDD data destruction would be melting (i.e. increasing the temperature of the platter to a point where atoms are no longer held in a lattice and are free to randomly roll around each other.)



We should also pose the question - how sensitive is my data? Consider that in the instance where the drive electronics have failed and all that needs replacing is a circuit board, your drive might be read by anyone from a low-level technician to someone who would pass your data to an international competitor, if one exists. Plan accordingly.



So if we are going to use HDDs as backup media, we have to make one of several choices. They are in order of least secure/easiest/cheapest -> most secure/hardest/most expensive.

  * Accept that your data may be read by your HDD vendor or one of the states through which your HDD passes after it is RMA'd. Don't bother with encryption.
  * Keep and destroy (See [#HDD\_Destruction](#HDD_Destruction.md)) the HDDs that fail and can't be written to. For drives with bad sectors, overwrite the drive multiple times with random data. Accept that the bad sectors may be read, and that perhaps there is technology to figure out what was originally written to the drive. Also consider that the random data you write might not be [cryptographically secure](http://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator).
  * Encrypt the drive and store the key somewhere not on the drive. When the HDD fails, optionally write with random data if possible. Send the encrypted drive to be RMA'd, and hope that they don't have the technology to break the encryption. (See [#Cryptanalysis](#Cryptanalysis.md)) Encrypting your backup pools can also reduce the amount of trust you need to extend to the off-site storage locations, and mitigates risk in the event of theft. (See [#Encryption\_Caution](#Encryption_Caution.md))
  * Select reliable drives as best we can. Subject all HDDs to a period of burn-in with unimportant data (e.g. zeros, ones, /dev/random) to catch drives that are likely to fail early, and so decrease the number of production drives that will need to be RMA'd. Perhaps there are also some other practices that will also lead to increased drive reliability. Finally, when drives fail, eat the cost. Store the drives on site indefinitely, or optionally destroy the platters.

Do note that you are already making one of these choices with the drives on your servers, but you may not have consciously thought about it in those terms.



Also note that if past history is any guide, HDDs drop in market value rapidly. If the drive can last a few years before RMA, in terms of market value, the GB held by that drive probably approaches the cost sending the drive off for RMA. Even if capacity reaches a finite limit, as research costs are paid for prices of HDDs will drop to the cost of manufacture, which is significantly lower than the price of the lowest $/GB drives. Thus, the most secure option (e.g. not using RMA) seems to be a lot more expensive than it really is. If HDDs are failing, it is still potentially worthwhile sending a message to the manufacturer by rating the drives on an online website (e.g. newegg, amazon).




## Comparison with Tape ##

I'll start with a cost comparison, as that is the most exciting. Figures are from Newegg.com.


### Cost: ###

**Tape:** LTO 5, 1.5 TB tapes are around $75. Tape drives are $2350 minimum. If we have 6TB of data, we need $300 of tapes per backup set. To store a GFS setup, we need minimum 23 tape sets for a year. This works out to be $300 x 23 + $2350 = **$9250** minimum + $300 per year. If somehow the tape can store snapshots, we might only need 7 tapes. This would be **$4450.** Actually it would be higher than that, because eventually the next generation of tape drive would need to be bought as well.



**HDD**: To store 6TB of data, 1.5TB drives in a 6 drive RAIDZ2 require 6 x $70 = $420/backup set. Buy a camera bag for them on ebay at $60 and silicone covers for 6x$4, brings this to $504. However, to achieve a similar level of archival ability using snapshots, we only really need maybe 4 backup sets in daily rotation + another 2 or 3 that are backed up to only every so often to guard against the unlikely event of something inadvertently and silently destroying your regular rotation set. So 7 backup pools at $504 x 7 = $3528. If we add the cost of $120 for 3 dual HDD docks and $10 for the two SATA to e-SATA cables poking through the back of your machine (or $30 if you want to get fancy and use a genuine dual e-SATA back plate in addition to the one that should be standard on your case), that's still only $3528 + $150 = **$3678 + $700 per year.** Do note that over this time some HDDs will be expected to die, but most will be replaced under warranty. Also note that the $700 per year figure is likely not that high because you can use old server HDDs in backup pools.



HDD based backup has potential to be cheaper in most cases, and since there is no need to buy an expensive tape drive, we can trade off money against data security as the organization grows. For example, if we are willing to use multiple RAIDZ, or if we have only a small amount of data, we can get by with a system costing less than a thousand dollars. As the data requirement grows we just buy more HDDs without necessitating a jump to a more expensive tape drive.


### Comparison with Tape Features ###

I will go through each “feature" bullet point of tape, and compare with HDD.

  * the possibility to recover data that has been destroyed either intentionally or unintentionally, going back as long as the organization exists.

This is achieved through keeping snapshots in the same fashion. It's just that the snapshots are not tied to physical media. Your data can survive in much the same way that genetic information encoded in error-correcting DNA is transmitted through multiple generations of higher order but very finite organisms, without needing an "archival quality" organism.

  * a resilient archival format that lasts 20+ years, with in-built error correction.

See above. Since ZFS hashes allow self-healing when there is redundancy in the pool, you can both know that your data is sound and have it be fixed when there are problems with the underlying media. And when the media gets old, you transfer it with the same end to end security that what was read is what was written.

  * due to tape resiliency, we can effectively ignore issues of RMA and security. Any bad tapes you can keep on hand without much cost.

This can be managed, though with costs. However, unless your organization has a secure plan for what to do with failed drives that have been used in production, there may be no real difference between tape backups and HDD backups, since a chain is only as strong as its weakest link.

  * some protection against data corruption in your main storage array, again, going back years

ZFS hashes will prevent most data corruption from occurring in the first place, and if there is not enough redundancy to allow self-healing, a “# zpool scrub" will notify the admin that there is a problem and hence a restore needs to be attempted from a backup pool.

  * protection against disk failure

See above.

  * mitigation of data loss in the case that your system gets destroyed or compromised - i.e. capable of off-site, offline storage. Data should be able to be restored in full from some date before the date the system was compromised.

This is no different from HDD based pool to tape. The HDD based method does require that some special care is taken to keep a couple backups that are only backed up to infrequently, to mitigate the risk of a compromise or mistake that is not caught in time.

  * resources geared towards protecting data that is most likely wanting to be recovered, e.g. someone deletes a file and they want the newest version within a week.

The GFS system is the same, so the same protection is afforded.

  * physically quite robust, probably not very fireproof but quite shock resistant.

This can be mitigated through the use of silicone HDD cases and padded bags, not to mention redundancy in the backup pool to allow the destruction of a HDD every now and then, provided that it is promptly replaced.

  * speed in writing the backup, as only the increment is transferred.

This is the case with zxfer, as it uses zfs send/receive and hence sends only the new snapshots which just hold the difference in the data.

  * historical snapshots of data usually aren't destroyed by a mistake.

It is possible that if a flaw in your snapshot management tool (or a mistaken sysadmin) causes snapshots on the main system to be deleted, they might be deleted on the destination as typically needs to be done in order to sync the two. That is why zxfer has the ability to protect “grandfather" snapshots from being deleted, by specifying that only snapshots that are a certain number of days old can be deleted.


### Comparison with Tape Disadvantages ###

Here is where HDD based pools shine - eliminating many of the disadvantages of tape.

  * Recovery is tedious. While the speed at which data can be written to tape is quite fast, random access is very slow and inconvenient. It's very much like trying to get to your favorite part of a movie on a VHS tape - it takes time.

Typical recovery (e.g. someone deletes a file and later wants it back) is very fast and convenient. Entire system recovery should be similar in speed, if not faster.

  * Initial cost is high and not suited to organic growth.

Initial cost is nearly as low as your budget, with the vast majority of the expense being yearly maintenance (i.e. purchase of HDD).

  * You may not be transferring snapshots to the tape. In this case, GFS as outlined above will require you to keep about 7+4+12 = (23 + N) x the size of your data set, where N is the number of years you have kept.

In this case, the HDD pool based method wins big, since less copies are used.



Another area where ZFS shines is in snapshotting. Since it is highly desirable that a backup represents a consistent “snapshot" of your data, the fact that ZFS can do this atomically across an entire pool is a godsend. It is also a natural fit that it is these snapshots that are sent when backing up, and also that by only sending the snapshots not present on the backup (i.e. only the delta), we can backup a system in a minimum of time.


### Other Discussion ###

There are some situations where tapes are still superior. Tapes are the go-to choice in systems where:

  * very large amounts of data are produced at a continuous rate (e.g. PB/year), and
  * this data is not used on a day to day or even year to year basis, but is likely to be used again in the future.

The cardinal example of something like this is the Large Hadron Collider; massive amounts of measurement data are taken. Years later, someone comes up with a credible theory that can be proven or disproven by a past experiment. In that case, it is a simple matter of pulling all the data from the tapes corresponding to that particular experiment.

Another reason for tapes - for a cautious and moderately financed business owner or CEO, tapes can also make sense as a periodic archival mechanism that would be kept in addition to HDD pool backups in a place only they have access to. This does place an upper limit on the damage an employee can do, and removes the  ability of the employee to go back in time and doctor records so as to cover their tracks. (This is provided that the owner actually verifies that the tapes he receives hold all the data they are supposed to.)

If there is a catch, it is that the zxfer project is young, and that the data must be stored on FreeBSD, Solaris, or a future operating system running ZFS such as Illumos. However, ZFS is mature and has a  technological lead on competing file systems. Prudent organizations like to let other people be the guinea pigs for experiments like file systems. That means that at least for a few years, if you want to have low-cost, reliable and convenient backups using HDDs, the answer lies with ZFS. Fortunately it is a lot easier for a Linux admin to migrate to a Unix such as FreeBSD or Solaris than it is to migrate to the Microsoft world.


## Conclusion ##

HDD based backups are a best fit for entities with a moderate amount of data that is slowly added to all the time. For many if not the majority of use cases, an organization or individual now has the technology at hand to either increase the reliability and convenience of backups, lower the cost, or both. Additionally, a HDD based solution is scalable without large jumps in cost. This applies for any entity from individual through to a large corporation provided that the data accumulation pattern of the organization is a fit with this system.


---


## Footnotes ##

### GFS ###
At the present time, it seems like ZFS Auto-Snapshot Service and Time-Slider in Solaris, and zfs-snapshot-mgmt from FreeBSD ports are the best tools to do this.

### Rogue Sysadmin ###
A rogue sysadmin who completely trashes an organization's data and all its backups would be somewhat infamous, and likely to never be employed again by a company in a role having anything to do with computers. Technically, it might be possible that they make enough illicit money in the process to make it still worth considering, but there are a lot of costs in leaving the country, not just financial. Not to mention the risk in getting caught.

### HDD Destruction ###
Considering that even a tiny scrap of HDD can have GB of data on it and that it can be read with an electron microscope, HDDs would need to either be reduced to dust, melted, or nuked from orbit as it is the only way to be sure. Actually, melting should be sufficient. HDD platters are either aluminum or glass (Hitachi).  [Backyard metal casting](http://www.backyardmetalcasting.com) provides several methods for somewhat inexpensively turning your Al platters into ingots. A household microwave and “microwave kiln" kits provide relatively inexpensive options for glass platters.

### Cryptanalysis ###
Whenever someone presents arguments about how difficult it is to break a given form of encryption (other than a One Time Pad), I am always reminded of how indecipherable Enigma was supposed to be, and how many years the secret of the decryption was able to be kept from the public despite multiple people obviously having worked on that problem. I also think about the speed advantages that specialized hardware has over a general CPU.

Also it will almost certainly not be manifestly obvious that the encryption you use is broken and that your data is being read. Anyone sufficiently skilled and funded to break the encryption can be assumed to have the smarts to manufacture a plausible explanation as to why they know something they shouldn't, in the same way that the Allies would visibly fly scout planes over Axis formations they had prior knowledge of through cracked encrypted information.

### Encryption Caution ###
Do note that if you are going to use encryption, I cannot stress enough how important it is to make multiple backups of your encryption keys and pass phrases. It is also worthwhile having an unencrypted backup or two. If you inadvertently lock yourself out of your data and have irretrievably lost the key, there shall be much wailing and gnashing of teeth.
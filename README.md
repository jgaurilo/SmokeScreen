## NOTICE ##
As of 18-MAY-2017, rclone is currently banned from connecting to ACD.

# SmokeScreen
Host your Plex media with (or without) rclone-crypt on cloud storage, and a local "fake" cache for Sonarr and Radarr to scan to prevent Google Drive API limit bans. Special thanks to Reddit users /u/gesis and /u/ryanm91 for most of the heavy lifting.

# Pre-requisites
This project relies on:
* rclone
* unionfs-fuse
* Plex Media Server
* Plex running as the same user as the scripts
* Sonarr, Radarr, SABnzbd, Torrent Clients (optional)

# Installation
clone the repo then
* move smokescreen.conf to ~/.config/SmokeScreen/smokescreen.conf
* move the scripts to ~/bin/

# Required rclone Remote
    
Create a remote in rclone that points at the top level of your Cloud Storage Provider. To use crypt, two remotes are required, with one being a crypt remote pointing at the one configured for your cloud storage. Read the documentation at rclone for further information.

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Cloud Storage is mounted you should see this file at `$mediadir/$checkfilename`. In the default configuration this file will be located at `~/content/google-check`. Use rclone to upload a file of this name to your Google drive's `$cloudsubdir`. `Media` is the default top level subfolder on cloud storage in the configuration.
    
For example:

    touch ~/google-check
    rclone copy ~/google-check GSUITE:Media

Now mount the system by running the `mount.remote` script. You should see your Google drive mounted at ~/.google and you should see a union of your local media and Google media directory at ~/content. If you don't, stop here and resolve it before continuing.

# CRON
CRON is used to automatically mount the drives, upload content to cloud storage, scan new media in to Plex, keep the cache up to date and remove local copies of media.

Add the following to your user's crontab:

    @hourly /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    @hourly /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1 
    @hourly /home/USER/bin/nuke.local >> /home/USER/logs/nuke.local.log 2>&1
    */2 * * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1

# Media Configuration
All media that is saved to `$localmedia` is uploaded to cloud storage and removed locally.

The configuration file and scripts outline certain configuration requirements for the additional software. The short version is:

* Plex should look at `$plex_shows_folder` for TV Series,`$plex_movie_folder` for Movies and `$plex_music_folder` for Music
* Sonarr series root should be `$localcache/$media_shows` if using the cache, or `$plex_shows_folder` if not
* Radarr movies root should be `$localcache/$media_movie` if using the cache, or `$plex_movie_folder` if not

# Plex Media Server Specifics
When using SmokeScreen, PMS must be configured to no longer scan for media, and disable media analysis. If you leave these options enabled, you will likely end up with a 24H ban for hitting Google's API too much. Since rclone provides no caching of data from your cloud service, each request hits the API instead of a local cache like other mount solution.

Specifically, you must disable:
* Settings -> Library -> Update my library automatically
* Settings -> Library -> Run a partial scan when changes are detected
* Settings -> Library -> Include music libraries in automatic updates
* Settings -> Library -> Update my library periodically
* Settings -> Scheduled Tasks -> Update all libraries during maintenance
* Settings -> Scheduled Tasks -> Upgrade media analysis during maintenance
* Settings -> Scheduled Tasks -> Perform extensive media analysis during maintenance

It is also recommended to disable:
* Settings -> Library -> Empty trash automatically after every scan
* Settings -> Library -> Allow media deletion

# Sonarr and Radarr Configuration
Set the two variables in the configuration `sonarrenabled` and `radarrenabled` = 1 if you would like to enable the cache. Once saved, run `scan.media` to build the local cache. Once it's complete, continue by reconfiguring Sonarr and Radarr to look at ~/.localmedia-cache as their root folder instead of wherever you had them pointed before and on the 'Connect' tab of the settings pages, add a custom script that points at `sonarr.cache` in Sonarr on `Download` and `Upgrade` and one in Radarr that points at `radarr.cache` that notifies on `Download` and `Upgrade`.

Media that these tools download will follow the following path:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to the .localmedia-cache folder
* The custom script in Sonarr/Radarr will move the file to the .localmedia folder
* update.cloud will then upload it to cloud storage and remove it from .localmedia

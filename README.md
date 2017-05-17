# SmokeScreen
Host your Plex media with (or without) rclone-crypt on Google Drive with cold storage on ACD, and a local "fake" cache for Sonarr and Radarr to scan to prevent Google Drive API limit bans. Special thanks to Reddit users /u/gesis and /u/ryanm91 for most of the heavy lifting.

# Pre-requisites
This project relies on:
* rclone
* unionfs-fuse
* Plex Media Server
* Sonarr, Radarr, SABnzbd, Torrent Clients (optional)

# Installation
clone the repo locally, and then
* move smokescreen.conf to ~/.config/SmokeScreen/smokescreen.conf
* move the scripts to ~/bin/
* Edit smokescreen.conf to your environment
  
# Configuration
The smokescreen.conf is well documented with how things must be configured. The `sonarr.cache` and `radarr.cache` files outline the configuration requirements in those pieces of software.

# Preparation
There are a number of things required:
* A local folder where media will be saved
* A local folder where the cache will be built (Optional - for use with Sonarr/Radarr)
* A local mount point for Google Drive
* A local mount point for the union of Google Drive and Local Media folders

# Required Local Folders
Note: these match the default configuration supplied

    mkdir ~/.localmedia
    mkdir ~/.localmedia-cache
    mkdir ~/.google
    mkdir ~/media

# Required rclone Remotes

    rclone config
    
You should create only the remotes you require. The "default" configuration is unencrypted data on Google and encrypted data on ACD, so three remotes are required:

* ACD: remote pointing at the top level of your Amazon Cloud Drive
* ACDCRYPT: rclone-crypt remote pointing at ACD:
* GSUITE: remote pointing at the top level of your Google Drive

If `$encryptgsuite` is set to 1 a fourth remote is required:

* GSUITECRYPT: rclone-crypt remote pointing at GSUITE:

If `$encryptamazon` is set to anything other than 1, `ACDCRYPT:` is not required.

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Google is mounted you should see this file at `$mediadir/$checkfilename`. In the default configuration this file will be located at `~/media/google-check`. Use rclone to upload a file of this name to your Google drive's `$cloudsubdir`. `Media` is the default top level subfolder.

If using encryption for Google:

    touch ~/google-check
    rclone copy ~/google-check GSUITECRYPT:Media
    
If not using encryption for Google:

    touch ~/google-check
    rclone copy ~/google-check GSUITE:Media

# CRON Jobs To Get It All Working
Add the following to your user's crontab:

    # Every five minutes, attempt to upload new media to the cloud
    */5 * * * * /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    # Every day at 1am kick off a Plex media scan and backup newly downloaded music to the cloud
    0   1 * * * /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1 
    # Every two minutes, check to make sure Google is still mounted
    */2 * * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1
    
# Sonarr/Radarr/Plex Configuration
The scripts outline certain configuration requirements for the additional software. The short version is:

* Plex should look at `$mediadir/$media_shows` for TV Series
* Plex should look at `$mediadir/$media_movie` for Movies
* Plex should look at `$mediadir/$media_music` for Music (See note)
* Sonarr should look at `$localcache/$media_shows` as its root folder
* Radarr should look at `$localcache/$media_movie` as its root folder

Note: For proper music handling, the `$otherunion` variable should be set so that your Music folder appears as a folder underneath the unionfs mount `$mediadir` and Plex pointed at it instead of your local Music folder. The scripts explicitly scan new media found there and nowhere else. Feel free to modify the `scan.media` script to accomodate your configuration if you do not wish to include your music folder in the unionfs mount.

## NOTICE ##
As of 18-MAY-2017, rclone is currently banned from connecting to ACD.

# SmokeScreen
Host your Plex media with (or without) rclone-crypt on cloud storage with a primary and secondary cloud storage provider, and a local "fake" cache for Sonarr and Radarr to scan to prevent Google Drive API limit bans. Special thanks to Reddit users /u/gesis and /u/ryanm91 for most of the heavy lifting.

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
  
# Main Configuration
The configuration file `smokescreen.conf` is well documented with how things must be configured.

# Required Local Folders
Note: these match the default configuration supplied, feel free to use your own folders anywhere on your system.

    mkdir ~/.localmedia
    mkdir ~/.localmedia-cache
    mkdir ~/.google
    mkdir ~/media

If you still have local media on your drive, you can move it to ~/.localmedia or you can cahnge the configuration to point to where your local media is actually stored. If you don't have any media locally, the defaults will work well.

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
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Google is mounted you should see this file at `$mediadir/$checkfilename`. In the default configuration this file will be located at `~/media/google-check`. Use rclone to upload a file of this name to your Google drive's `$cloudsubdir`. `Media` is the default top level subfolder on cloud storage in the configuration.

If using encryption for Google:

    touch ~/google-check
    rclone copy ~/google-check GSUITECRYPT:Media
    
If not using encryption for Google:

    touch ~/google-check
    rclone copy ~/google-check GSUITE:Media

Now mount the system by running the `mount.remote` script. You should see your Google drive mounted at ~/.google and you should see a union of your local media and Google at ~/media. If you don't, stop here and resolve it before continuing.

# Sonarr and Radarr Configuration
The `sonarr.cache` and `radarr.cache` files outline the configuration requirements for Sonarr/Radarr if you wish to use them. On the 'Connect' tab of the settings page, add a custom script that points at `sonarr.cache` in Sonarr on `Download` and `Upgrade` and one in Radarr that points at `radarr.cache` that notifies on `Download` and `Upgrade`. Scanning in to Plex will NOT occur without this.

Now run `scan.media initialize` to build the local cache. Leaving off the `initialize` argument will cause it to also attempt to upload music found in `$local_music_folder`. Once it's complete, continue by reconfiguring Sonarr and Radarr to look at ~/.localmedia-cache as their root folder instead of wherever you had them pointed before. Media that these tools download will follow the path of:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to the .localmedia-cache folder
* The custom script in Sonarr/Radarr will move the file to the .localmedia folder
* update.cloud will then upload it to cloud storage and remove it from .localmedia

# CRON
Add the following to your user's crontab:

    # Every five minutes, attempt to upload new media to the cloud
    */5 * * * * /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    # Every day at 1am kick off a Plex media scan and backup newly downloaded music to the cloud
    0   1 * * * /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1 
    # Every two minutes, check to make sure Google is still mounted
    */2 * * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1

# Media Configuration
All media that is saved to `$localmedia/` is uploaded to cloud storage and removed locally. All MUSIC stored at `$local_music_folder` is uploaded to cloud storage as a backup only, as Plex does not work well with music on cloud storage. Subfolders under `$localmedia` should match `$media_shows` and `$media_movie` for TV shows and movies.

The configuration file and scripts outline certain configuration requirements for the additional software. The short version is:

* Plex should look at `$mediadir/$media_shows` for TV Series
* Plex should look at `$mediadir/$media_movie` for Movies
* Plex should look at `$mediadir/$media_music` for Music (See note)
* Sonarr should look at `$localcache/$media_shows` as its root folder
* Radarr should look at `$localcache/$media_movie` as its root folder

Note: For proper music handling, the `$otherunion` variable should be set so that your Music folder appears as a folder underneath the unionfs mount `$mediadir` and Plex pointed at it instead of your local Music folder. The scripts explicitly scan new media found there and nowhere else. Feel free to modify the `scan.media` script to accomodate your configuration if you do not wish to include your music folder in the unionfs mount.

# Plex Media Server Specifics
When using SmokeScreen, PMS must be configured to no longer scan for media, and disable thumbnail generation and media analysis. If you leave these options enabled, you will likely end up with a 24H ban for hitting Google's API too much. Since rclone provides no caching of data from your cloud service, each request hits the API instead of a local cache like other mount solution.

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

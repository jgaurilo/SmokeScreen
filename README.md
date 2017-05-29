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
    mkdir ~/content

If you still have local media on your drive, you can move it to ~/.localmedia or you can cahnge the configuration to point to where your local media is actually stored. If you don't have any media locally, the defaults will work well.

# Required rclone Remotes

    rclone config
    
Create a remote that points at the top level of your Cloud Storage Provider

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Cloud Storage is mounted you should see this file at `$mediadir/$checkfilename`. In the default configuration this file will be located at `~/content/google-check`. Use rclone to upload a file of this name to your Google drive's `$cloudsubdir`. `Media` is the default top level subfolder on cloud storage in the configuration.
    
For example:

    touch ~/google-check
    rclone copy ~/google-check GSUITE:Media

Now mount the system by running the `mount.remote` script. You should see your Google drive mounted at ~/.google and you should see a union of your local media and Google at ~/media. If you don't, stop here and resolve it before continuing.

# Sonarr and Radarr Configuration
The `sonarr.cache` and `radarr.cache` files outline the configuration requirements for Sonarr/Radarr if you wish to use them. On the 'Connect' tab of the settings page, add a custom script that points at `sonarr.cache` in Sonarr on `Download` and `Upgrade` and one in Radarr that points at `radarr.cache` that notifies on `Download` and `Upgrade`. Scanning in to Plex will NOT occur without this.

Now run `scan.media` to build the local cache. Once it's complete, continue by reconfiguring Sonarr and Radarr to look at ~/.localmedia-cache as their root folder instead of wherever you had them pointed before. Media that these tools download will follow the path of:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to the .localmedia-cache folder
* The custom script in Sonarr/Radarr will move the file to the .localmedia folder
* update.cloud will then upload it to cloud storage and remove it from .localmedia

# CRON
Add the following to your user's crontab:

    @hourly /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    @hourly /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1 
    @hourly /home/USER/bin/nuke.local >> /home/USER/logs/nuke.local.log 2>&1
    */2 * * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1

# Media Configuration
All media that is saved to `$localmedia/` is uploaded to cloud storage and removed locally. Subfolders under `$localmedia` should match `$media_shows` and `$media_movie` for TV shows and movies, otherwise the Sonarr/Radarr caching will fail to work properly.

The configuration file and scripts outline certain configuration requirements for the additional software. The short version is:

* Plex should look at `$plex_shows_folder` for TV Series
* Plex should look at `$plex_movie_folder` for Movies
* Plex should look at `$plex_music_folder` for Music (See note)
* Sonarr series should look at `$localcache/$media_shows` as their root folder
* Radarr movies should look at `$localcache/$media_movie` as their root folder

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

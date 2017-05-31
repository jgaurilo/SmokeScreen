# SmokeScreen
Host your Plex media with (or without) rclone-crypt on cloud storage, and a local cache for Sonarr and Radarr to scan to prevent Google Drive API limit bans. Special thanks to Reddit users /u/gesis and /u/ryanm91 for most of the heavy lifting.

# Pre-requisites
This project relies on:
* rclone
* unionfs-fuse
* bc (sudo apt-get install bc)
* Plex Media Server
* Plex running as the same user as the scripts (VERY IMPORTANT)
* Sonarr, Radarr, SABnzbd, Torrent Clients (optional)

# Installation
clone the repo then
* move smokescreen.conf to ~/.config/SmokeScreen/smokescreen.conf
* move the scripts to ~/bin/

# Required rclone Remote
    
Create a remote in rclone that points at the top level of your Cloud Storage Provider. To use crypt, two remotes are required, with one being a crypt remote pointing at the one configured for your cloud storage. Read the documentation at rclone for further information. Set the configuration option `$primaryremote` to the remote you created in rclone.

# Default Configuration Variables

The default configuration creates folders and mount points in your user's home directory. This may not be acceptable for your configuration, so change them to a more suitable location. All steps in this README refer to the `$variable` name and not the `~/path` to avoid confusion.

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Cloud Storage is mounted you should see this file at `$mediadir/$checkfilename`. Use rclone to upload a file of this name to your Google drive's `$cloudsubdir`:

    touch ~/google-check
    rclone move ~/google-check GSUITE:Media

Now mount the system by running the `check.mount` script. You should see your Google drive mounted at `$googledir` and you should see a union of your local media and Google media directory at `$mediadir`. If you don't, stop here and resolve it before continuing.

# Plex Media Server Configuration
When using SmokeScreen, PMS must be configured to no longer scan for media, and disable media analysis. If you leave these options enabled, you will likely end up with a 24H ban for hitting Google's API too much. Since rclone provides no caching of data from your cloud service, each request hits the API.

Media libraries in Plex must be configured:
* Plex should look at `$plex_shows_folder`
* Plex should look at `$plex_movie_folder`
* Plex should look at `$plex_music_folder`

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

If you've created new libraries or modified the paths in existing libraries in Plex, cancel the scans the Plex initiates automatically. We will rescan everything once we're done configuring the other software.

# Sonarr and Radarr Configuration
To use Sonarr/Radarr without the local cache, set the two variables in the configuration `$sonarrenabled` and `$radarrenabled` = 0. Sonarr and Radarr should be configured to use `$mediadir/$media_shows` and `$mediadir/$media_movie` as the root folders for all series/movies. Be warned that excessive scanning by Sonarr/Radarr can lead to 24H bans by Google.

Media that these tools download will follow the following path with the cache disabled:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to $mediadir
* update.cloud will then upload it to cloud storage

To use the local cache, set the two variables in the configuration `$sonarrenabled` and `$radarrenabled` = 1. Reconfigure Sonarr and Radarr to look at `$localcache/$media_shows` and `$localcache/$media_movie` as their root folder. On the 'Connect' tab of the settings pages, add a custom script that points at `sonarr.cache` in Sonarr and `radarr.cache` in Radarr that notifies on `Download` and `Upgrade`.

Media that these tools download will follow the following path with the cache enabled:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to $localcache
* The custom script in Sonarr/Radarr will move the file to the $localmedia folder and scan the media in to Plex
* update.cloud will then upload it to cloud storage

If you are NOT using Sonarr or Radarr, set the configuration values for them to 0. Manually add Movies and TV series to `$localmedia/$media_shows` for TV and `$localmedia/$media_movie` for Radarr. Be sure that media placed here manually follows Plex's media naming conventions.

# Initial scan and cache build
Once all the software is configured correctly, run `scan.media 1` to force a complete scan of all media and build the cache for Sonarr and Radarr if it's enabled. Be prepared to wait, as this will take a while. It might be a good idea to run the scan in a screen so that if your connection is interrupted the scan won't stop: `screen /bin/bash ~/bin/scan.media 1`.

# Automatic Processing
CRON is used to automatically mount the drives, upload content to cloud storage, scan new media in to Plex, keep the cache up to date and remove local copies of media.

Add the following to your user's crontab:

    @hourly /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    @hourly /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1 
    @hourly /home/USER/bin/nuke.local >> /home/USER/logs/nuke.local.log 2>&1
    */2 * * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1

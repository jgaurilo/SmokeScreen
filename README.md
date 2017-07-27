# SmokeScreen
Host your Plex media with (or without) rclone-crypt on cloud storage with automated uploading and local cleanup of new media.

# Thank You
Reddit users /u/gesis and /u/ryanm91

# Pre-requisites
This project relies on:
* rclone 1.37+ (require the tpslimit option)
* unionfs-fuse
* bc (sudo apt-get install bc)
* Plex Media Server
* Plex running as the same user as the scripts (VERY IMPORTANT)
* Sonarr, Radarr, SABnzbd, Torrent Clients (optional)

# Installation
clone the repo then
* move smokescreen.conf to ~/.config/SmokeScreen/smokescreen.conf
* move the scripts to ~/bin/
* make sure all the scripts are executable

# Default Configuration Variables

The default configuration creates folders and mount points in your user's home directory. This may not be acceptable for your configuration, so change them to a more suitable location. All steps in this README refer to the `$variable` name and not the `~/path` to avoid confusion.

NOTE: `$cloudsubdir` MUST be set to a value. Setting it to an empty string so that media is placed at the very top level of your cloud storage is unsupported, and could have odd side-effects. Some of the provided scripts assume that this variable has a value, and as such may not work properly if omitted.

# Required rclone Remotes

## Without Encryption ##
Create a remote in rclone that points at the TOP LEVEL of your cloud storage provider (Do not enter a subfolder). Set the configuration option `$primaryremote` to the remote you created in rclone, and set the configuration option `$cloudsubdir` to a descriptive name.

`$cloudsubdir` will be created at the top level of your cloud storage automatically when `update.cloud` is run the first time, and media will appear in subfolders beneath it. When this remote is mounted, you will see a subfolder named `$cloudsubdir` at `$clouddir`.

## With Encryption ##
First create a remote in rclone that points at the TOP LEVEL of your cloud storage provider. Then create a second remote that is a `crypt` remote who's `remote` is the name of your unencrypted remote, followed by `:encrypted`. Follow the instructions to complete setting up the encrypted remote (entering passwords, etc). It is recommended to choose to encrypt filenames. Set the configuration option `$primaryremote` to the encrypted remote you created in rclone, and set the the configuration option `$cloudsubdir` to a descriptive name.

The first time `update.cloud` is run, a folder named `encrypted` will be created at the top level of your cloud storage, and a sub-folder with the encrypted value of `$cloudsubdir` will be created and media will appear encrypted in subfolders beneath it when browsing it via the web. When this remote is mounted with rclone, you will see a decrypted subfolder named `$cloudsubdir` at `$clouddir`.

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Cloud Storage is mounted you should see this file at `$mediadir/$checkfilename`. Use rclone to upload a file of this name to your cloud storage `$cloudsubdir` folder. Example:

    touch ~/google-check
    rclone move ~/google-check GSUITE:Media

Now mount the system by running the `check.mount` script. You should see your cloud storage mounted at `$clouddir` and you should see a union of `$localmedia` and `$clouddir/$cloudsubdir` at `$mediadir`. If you don't, stop here and resolve it before continuing.

# Plex Media Server Configuration
The newest (1.37 at time of this writing) version of rclone now includes options to limit API usage. While this means you can allow Plex to automatically scan your media libraries to keep them up to date, the `scan.media` script is still available so that you can schedule scans. Likewise, the `media.upgrade` script can be used by Sonarr/Radarr to scan newly downloaded episodes and movies in to Plex. These scripts are more reliable when using Plex with unionfs and cloud storage. They also allow scanning of large music libraries on systems where Plex's scanner fails to properly do so.

## Plex Scanning (Recommended Setup) ##
* Settings -> Library -> Disable Update my library automatically
* Settings -> Library -> Disable Run a partial scan when changes are detected
* Settings -> Library -> Disable Include music libraries in automatic updates
* Settings -> Library -> Disable Update my library periodically
* Settings -> Scheduled Tasks -> Disable Update all libraries during maintenance

## Plex Media Libraries ##
Media libraries in Plex must be configured:
* Plex should look at `$plex_shows_folder` for TV Series
* Plex should look at `$plex_movie_folder` for Movies
* Plex should look at `$plex_music_folder` for Music

## Additional Options (Recommended) ##
* Settings -> Library -> Disable Empty trash automatically after every scan
* Settings -> Library -> Disable Allow media deletion
* Settings -> Scheduled Tasks -> Disable Upgrade media analysis during maintenance
* Settings -> Scheduled Tasks -> Disable Perform extensive media analysis during maintenance

If you've created new libraries or modified the paths in existing libraries in Plex, cancel the scans that Plex initiates automatically. We will rescan everything once we're done configuring the other software.

# Sonarr and Radarr Configuration
Sonarr and Radarr should be configured to use `$mediadir/$media_shows` and `$mediadir/$media_movie` as the root folders for all series/movies. The included script `media.upgrade` can be used by Sonarr and Radarr to automatically remove old versions of files when Sonarr/Radarr import something that already exists and also initiate a Plex scan of newly added media. If you do not use this script, you will need to manually clean up unionfs metadata files and old versions of shows/movies from your cloud storage.

Media that these tools download will follow the path:

* Download client (SAB, Deluge, etc) downloads to temp directory
* Sonarr/Radarr "import" the file to `$mediadir`
* `update.cloud` will then upload it to cloud storage

If you are NOT using Sonarr or Radarr, manually add episodes and movies to `$localmedia/$media_shows` for TV and `$localmedia/$media_movie` for movies. Be sure that media manually placed here follows Plex's media naming expectations.

# Initial scan
Once all the software is configured correctly, run `scan.media 1` to force a complete scan of all media. Be prepared to wait, as this will take a while. It might be a good idea to run the scan in a screen so that if your connection is interrupted the scan won't stop: `screen /bin/bash ~/bin/scan.media 1`. Do not continue until this scan is complete.

# Automatic Processing
CRON is used to automatically mount the drives, upload content to cloud storage, scan new media in to Plex, and remove local copies of media.

Add the following to your user's crontab:

    @hourly       /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    0    1  * * * /home/USER/bin/scan.media   >> /home/USER/logs/scan.media.log 2>&1
    0    23 * * * /home/USER/bin/nuke.local   >> /home/USER/logs/nuke.local.log 2>&1
    */5  *  * * * /home/USER/bin/check.mount  >> /home/USER/logs/check.mount.log 2>&1

If you prefer to allow Plex to handle all media scanning, do not add the line for `scan.media` to your CRON. 

# A Note About Music
Since cloud storage-hosted music doesn't work well with Plex, but we may have disabled automatic scanning, newly added music might not appear automatically in Plex. The configuration variables `$plex_music_folder` and `$plex_music_category` are available so that `scan.media` will scan newly added music. Leave these variables blank if you do not use Plex for music, or if you have Plex set to automatically scan your music folder. Note, the `scan.media` script seems more reliable on larger music libraries, and using it is recommended over Plex's internal scanning regardless.

DO NOT store your music at `$localmedia` as it will get uploaded to cloud storage and deleted locally. Also, do not store your music in any of the other folders configured for FUSE mounts (`$clouddir`, `$mediadir`, etc). Use a local folder instead.

# Plex Media Scanner Utility Script
There is a script included called `pms`. This script just sets up the environment so that you can call the Plex media scanner easily. Usage is:

    ./pms "whatever you want to send to the media scanner"
    
Make SURE to wrap the command line with quotation marks, so that the entire thing is treated as argument one being passed in. You can use it to get your library IDs by calling it like this:

    ./pms "--list"
    
The SmokeScreen.conf file must be in place (and properly configured for your environment) before it will work.

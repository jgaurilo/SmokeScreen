# SmokeScreen
Host your Plex media with (or without) rclone-crypt on cloud storage. Special thanks to Reddit users /u/gesis and /u/ryanm91 for most of the heavy lifting.

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

# Required rclone Remotes
    
## Without Encryption: ##
Create a remote in rclone that points at the TOP LEVEL of your cloud storage provider. Set the configuration option `$primaryremote` to the remote you created in rclone, and set the configuration option `$cloudsubdir` to a descriptive name. This folder will be created at the top level of your cloud storage automatically when `update.cloud` is run the first time, and media will appear in subfolders beneath it.

## With Encryption ##
Create a remote following the `without encryption` instructions first. Then create a second remote that is a `crypt` remote who's `remote` is the name of your unencrypted remote, followed by `:encrypted`. Follow the instructions to complete setting up the encrypted remote (entering passwords, etc). It is recommended to choose to encrypt filenames. Set the configuration option `$primaryremote` to the encrypted remote you created in rclone, and set the the configuration option `$cloudsubdir` to a descriptive name.

The first time `update.cloud` is run, a folder named `encrypted` will be created at the top level of your cloud storage, and a sub-folder with the encrypted value of `$cloudsubdir` will be created and media will appear in subfolders beneath it.

# Default Configuration Variables

The default configuration creates folders and mount points in your user's home directory. This may not be acceptable for your configuration, so change them to a more suitable location. All steps in this README refer to the `$variable` name and not the `~/path` to avoid confusion.

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Cloud Storage is mounted you should see this file at `$mediadir/$checkfilename`. Use rclone to upload a file of this name to your cloud storage `$cloudsubdir` folder. Example:

    touch ~/google-check
    rclone move ~/google-check GSUITE:Media

Now mount the system by running the `check.mount` script. You should see your cloud storage mounted at `$clouddir` and you should see a union of your local media and cloud storage media directory at `$mediadir`. If you don't, stop here and resolve it before continuing.

# Plex Media Server Configuration
The newest (1.37 at time of this writing) version of rclone now includes options to limit API usage. This means we can enable Plex scanning without worrying about API bans.

Media libraries in Plex must be configured:
* Plex should look at `$plex_shows_folder` for TV Series
* Plex should look at `$plex_movie_folder` for Movies
* Plex should look at `$plex_music_folder` for Music

It is recommended to disable:
* Settings -> Library -> Empty trash automatically after every scan
* Settings -> Library -> Allow media deletion

If you've created new libraries or modified the paths in existing libraries in Plex, cancel the scans that Plex initiates automatically. We will rescan everything once we're done configuring the other software.

# Sonarr and Radarr Configuration
Sonarr and Radarr should be configured to use `$mediadir/$media_shows` and `$mediadir/$media_movie` as the root folders for all series/movies.

Media that these tools download will follow the following path with the cache disabled:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to $mediadir
* update.cloud will then upload it to cloud storage

If you are NOT using Sonarr or Radarr, manually add Movies and TV series to `$localmedia/$media_shows` for TV and `$localmedia/$media_movie` for movies. Be sure that media manually placed here follows Plex's media naming expectations.

# Initial scan
Once all the software is configured correctly, run `scan.media 1` to force a complete scan of all media. Be prepared to wait, as this will take a while. It might be a good idea to run the scan in a screen so that if your connection is interrupted the scan won't stop: `screen /bin/bash ~/bin/scan.media 1`.

# Automatic Processing
CRON is used to automatically mount the drives, upload content to cloud storage, scan new media in to Plex, and remove local copies of media.

Add the following to your user's crontab:

    @hourly /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    0    1  * * * /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1
    0    23 * * * /home/USER/bin/nuke.local >> /home/USER/logs/nuke.local.log 2>&1
    */5  *  * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1

# A Note About Music
Since cloud storage-hosted music doesn't work well with Plex, but we may have disabled automatic scanning, newly added music might not appear automatically in Plex. The configuration variables `$plex_music_folder` and `$plex_music_category` are available so that `scan.media` will scan newly added music. Leave these variables blank if you do not use Plex for music, or if you have Plex set to automatically scan your music folder.

# Utility Script
There is a script included called pms. This script just sets up the environment so that you can call the Plex media scanner easily. Usage is:

    ./pms "whatever you want to send to the media scanner"
    
Make SURE to wrap the command line with quotation marks, so that the entire thing is treated as argument one being passed in. You can use it to get your library IDs by calling it like this:

    ./pms "--list"
    
The SmokeScreen.conf file must be in place (and properly configured for your environment) before it will work.

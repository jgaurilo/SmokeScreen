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

Let's get started:

    mkdir ~/.localmedia
    mkdir ~/.localmedia-cache
    mkdir ~/.google
    mkdir ~/media

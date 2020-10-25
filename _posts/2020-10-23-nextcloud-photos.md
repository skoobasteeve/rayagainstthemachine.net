---
layout: single
title:  "Better Nextcloud Photo Albums with Preview Generator and Exiftool"
date:   2020-10-24 19:00:00
categories: [Linux Administration]
tags: linux ubuntu nextcloud photos exiftool
comments: true
---

If you host and use a Nextcloud server, you know that it's good at many things. Unfortunately, photos are not one of its strengths. Since Nextcloud does not read photo metadata, your albums will often appear out-of-order. You may also notice that thumbnails and previews take a very long time to load and flipping quickly through a bunch of photos becomes a painful waiting game. 

The good news is that because Nextcloud is a wonderful piece of FOSS that we're self-hosting, we can make some modifications to smooth out these pain points. There are (2) pieces of software we'll be using to accomplish this:

- [Preview Generator](https://apps.nextcloud.com/apps/previewgenerator) (Nextcloud app)
- [exiftool](https://exiftool.org/) (Linux app)

Let's dive in!

# Previews and Thumbnails

First we need to fix our preview generation. By default, Nextcloud generates photo previews and thumbnails on-demand, leading to slow load times. To fix this, we're going to use the Preview Generator app for Nextcloud to pre-generate these previews on a regular basis. That way, your photos are ready to view as soon as you open the folder. 

1. **Install the Preview Generator app for Nextcloud**. From a Nextcloud account with admin permissions, navigate to the Apps section and locate Preview Generator under the Multimedia category. Click Download and enable. 

2. **Configure preview and thumbnail settings.** While the default settings work well from a performance standpoint, they cause Nextcloud to generate a huge number of previews and thumbnails for each photo. Once you add a lot of photos, you'll notice that these previews eat into your storage significantly. Fortunately, Preview Generator is highly configurable.
   
   **Note:** I've found that the below settings provide a good balance of resolution, performance, and storage usage for my environment, but you can tweak them depending on your needs.
   
   Choose default thumbnail sizes:
   
   ```bash
   sudo -u www-data php /var/www/nextcloud/occ config:app:set --value="32 256" previewgenerator squareSizes
   sudo -u www-data php /var/www/nextcloud/occ config:app:set --value="256 384" previewgenerator widthSizes
   sudo -u www-data php /var/www/nextcloud/occ config:app:set --value="256" previewgenerator heightSizes
   ```
   
   Next, edit your config.php to specify the maximum preview size for images. This is going to effect the appearance and load time of images when you click on them.
   
   ```bash
   sudo nano /var/www/nextcloud/config/config.php
   ```
   
   Find the below lines toward the end of the file. If they don't exist, add them to the block. 
   
   ```php
   'preview_max_x' => '2048',
   'preview_max_y' => '2048',
   'jpeg_quality' => '60',
   ```
   
   Save the file and restart the web server.
   
   ```bash
   sudo service apache2 restart
   ```
   
   

3. **Generate initial previews.** Open a terminal on your Nextcloud host machine and run the following command:
   
   ```bash
   sudo -u www-data php /var/www/nextcloud/occ preview:generate-all -vvv
   ```
   
   The above command is run as the web server user since they are typically the owner of the Nextcloud directory, though you may need to tweak it based on your server configuration.
   
   **NOTE:** Depending on the amount of photos you have, this could take a while to complete and use a high amount of resources. If you have users beyond yourself, it's probably best to run it during a low-activity period. 

4. **Add a cron job.** To avoid problems, you should edit the crontab of the web server user:
   
   ```bash
   sudo -u www-data crontab -e
   ```
   
   Add the following below line to the file. The below example runs the command every 10 minutes, but you can set the frequency to whatever you want.
   
   ```bash
   * /10 * * * * /usr/bin/php -f /var/www/nextcloud/occ preview:pre-generate
   ```
   
   **NOTE:** If you have a specific version of PHP installed beyond your distro's default, you'll want to specify that version of the binary above (e.g. `php7.4` instead of `php`).



# Photo Sorting

Sort order is a bit more difficult to tackle. Nextcloud is first and foremost a file sharing application, so it views files just as your file system would. This is great until you get to photo albums.

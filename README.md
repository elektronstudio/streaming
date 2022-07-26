## Usage

### Input

```
rtmp://streaming.elektron.art/live/some-key-here
```

Example with streamkey `stream1` playing the file `tron.mp4`

```
ffmpeg -stream_loop -1 -re -i tron.mp4 -c:v libx264 -s 852x480 -b:v 128K -crf 19 -c:a aac -ar 22050 -ac 1 -f flv rtmp://streaming.elektron.art/live/stream1
```

### Output 


```
http://streaming.elektron.art/hls/some-key-here.m3u8
```

Example with streamkey `stream1`

```
http://streaming.elektron.art/hls/stream1.m3u8
```

## Installation

> TODO: transcoding

Installation follows roughly these tutorials:

https://www.digitalocean.com/community/tutorials/how-to-set-up-a-video-streaming-server-using-nginx-rtmp-on-ubuntu-20-04

https://simplebackups.com/blog/mounting-digitalocean-spaces-and-access-bucket-from-droplet/

### VPS

Create a VPS with Ubuntu 20

### Nginx

```
sudo apt update
sudo apt install -y nginx
sudo apt install -y libnginx-mod-rtmp
sudo apt install -y ffmpeg
```

Run

```
sudo nano etc/nginx/nginx.conf
```

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	##
	# SSL Settings
	##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	##
	# Gzip Settings
	##

	gzip on;

	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}

rtmp {
        server {
                listen 1935;
                chunk_size 4096;
                allow publish all;
		allow play all;
				
                application live {
                        live on;
			hls on;
                        hls_path /var/www/html/stream/hls;
                        hls_fragment 3;
                        hls_playlist_length 60;
			record all;  
			record_path /tmp/records/;  
			record_suffix ___%y_%m_%d__%H_%M_%S.flv;  
			exec_record_done sudo /tmp/records/records.sh $path $basename;
                }
        }
}
```

Run

```
sudo nano /etc/nginx/sites-available/rmtp
```

```nginx
server {
    listen 80;
    server_name localhost;
    location / {
        add_header Access-Control-Allow-Origin *;
        root /var/www/html/stream;
    }
    location /stat {
        rtmp_stat all;
        rtmp_stat_stylesheet stat.xsl;
    }
    location /stat.xsl {
        root /var/www/html/rtmp;
    }
    # rtmp control
    location /control {
        rtmp_control all;
    }
}

types {
    mpd;
}
```

and then

```
sudo ln -s /etc/nginx/sites-available/rtmp /etc/nginx/sites-enabled/rtmp
sudo mkdir /var/www/html/stream
rm /etc/nginx/sites-available/default
```

### Stats

```
sudo mkdir /var/www/html/rtmp
sudo cp /usr/share/doc/libnginx-mod-rtmp/examples/stat.xsl /var/www/html/rtmp/stat.xsl
```

### Recording

```
mkdir /var/records
touch /var/records/records.sh
chmod +x /var/records/records.sh
sudo chown -R www-data:www-data /var/records
```

Run

`nano /var/records/records.sh`:

```sh
#!/bin/bash 
ffmpeg -i $1 -c copy /var/records/$2.mp4;
duration=`ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 /var/records/$2.mp4`
cp /var/records/$2.mp4 /media/elektron/records/$2___$duration.mp4;
rm /var/records/$2.flv
rm /var/records/$2.mp4
```

Then

```
visudo
```

and paste in the follwing:

```
www-data ALL=NOPASSWD: /tmp/records/records.sh
```

### S3


```sh
sudo apt install s3fs
# Replace ACCESS_KEY_ID:SECRET_ACCESS_KEY
echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ${HOME}/.passwd-s3fs && chmod 600 ${HOME}/.passwd-s3fs
mkdir -p /media/elektron
s3fs elektron /media/elektron -o passwd_file=${HOME}/.passwd-s3fs -o url=https://fra1.digitaloceanspaces.com -o use_path_request_style -o default_acl=public-read-write -o umask=0000,mp_umask=0000,uid=33,gid=33 -o nonempty
mkdir -p /media/elektron/records
```

#### Finishing

```
sudo systemctl reload nginx.service
```

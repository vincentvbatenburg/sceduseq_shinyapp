# How to set up a shinyapp
based on [source](https://www.charlesbordet.com/en/guide-shiny-aws/#how-to-install-shiny-server)

### AWS setup
- AMI = Ubuntu 20 *(certbot has no explicit support for ubuntu 22)*
- Architecture = x86
- Instance type = t2.small *(t2.micro is not enough to install the shiny package)*
- Storage = 22GiB 

### Libraries and dependencies for neccesary r packages
```
sudo apt update -qq
sudo apt install build-essential
sudo apt install zlib1g-dev
```

### Add cran to apt and install r
```
sudo apt install --no-install-recommends software-properties-common dirmngr
wget -qO- https://cloud.r-project.org/bin/linux/ubuntu/marutter_pubkey.asc | sudo tee -a /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc
sudo add-apt-repository "deb https://cloud.r-project.org/bin/linux/ubuntu $(lsb_release -cs)-cran40/"
sudo apt install --no-install-recommends r-base
```
based on [source](https://cloud.r-project.org/bin/linux/ubuntu/)

### Instal shiny r package
```
sudo su - -c "R -e \"install.packages('shiny', repos='https://cran.rstudio.com/')\""
```

*this way of installing is to make sure packages are installed for all users, including the shiny user which runs the shiny app*

```
sudo apt-get install gdebi-core
wget https://download3.rstudio.org/ubuntu-18.04/x86_64/shiny-server-1.5.21.1012-amd64.deb
sudo gdebi shiny-server-1.5.21.1012-amd64.deb
```
based on [source](https://posit.co/download/shiny-server/)

### Open a port on the firewall on AWS so the app can be accessed (can be found under inbound security)
- type = Custom TCP Rule
- port = 3838

> [!NOTE]
> By now you can access the default shiny app on [IPADRESS]:3838


### Rsync the app to the server
```
rsync -av --progress -e "ssh -i [MYKEY].pem" app.R ubuntu@[MYSERVER]:/home/ubuntu/[MYFOLDER]
```

### Make a link to the app in the shiny server folder where shiny looks for apps
```
cd /srv/shiny-server
sudo ln -s ~/[MYFOLDER] .
```

### Edit the shiny conf file 
```
sudo vim /etc/shiny-server/shiny-server.conf
```
- add `preserve_logs true;` to the top 
- change `directory_index on;` to `directory_index off;`
- change `site_dir` to `app_dir`
- add the correct location of the app

### Install additional R packages needed for the app
```
sudo su - -c "R -e \"install.packages(c('data.table', 'magrittr', 'ggplot2', 'stringr', 'patchwork', 'R.utils'), repos='https://cran.rstudio.com/')\""
```

### Restart the shiny server
```
sudo systemctl reload shiny-server
```

> [!NOTE]
> Before continuing setup a domain name at a domain name registrar, for example godaddy

### Install and setup nginx
```
sudo apt install nginx
```

make a config file for the app so nginx will pick it up
```
cd /etc/nginx/sites-available
sudo vim shiny.conf
```
fill in the `default nginx shiny setup` ***see bottom of this page***

```
cd ../sites-enabled
sudo ln -s ../sites-available/shiny.conf .
sudo nginx -t
```

most likely nginx will say there are settings missing, if it says all ok than skip the next steps and restart

```
sudo vim /etc/nginx/nginx.conf
```
add `snippet` to the nginx config ***see bottom of this page***
```
sudo nginx -t
```
now the message should say ok

```
sudo systemctl restart nginx
```

### Open a port on the firewall on AWS for nginx (can be found under inbound security)

- type = Custom TCP Rule
- port = 80

### Install certbot
```
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx --register-unsafely-without-email
```
based on [source](https://certbot.eff.org/instructions)

### Open a port on the firewall on AWS for https (can be found under inbound security)

- type = Custom TCP Rule
- port = 443

## default nginx shiny setup
```
server {
    # listen 80 means the Nginx server listens on the 80 port.
    listen 80;
    listen [::]:80;
    # Replace it with your (sub)domain name.
    server_name [MYDOMAIN];
    # The reverse proxy, keep this unchanged:
    location / {
        proxy_pass http://localhost:3838;
        proxy_redirect http://localhost:3838/ $scheme://$host/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 20d;
        proxy_buffering off;
    }
}
```

## snippet
```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

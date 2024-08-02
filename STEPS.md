# Deploy multiple Django applications on the VPS server using nginx and gunicorn

### _Prerequisite_
1. Webuzo or any Control Panel to complete step 3 & 4
2. Ubuntu 22.04
3. Create the domain
4. Create the MySQL database and connect it with the user 

Here I will show you how you can deploy multiple Django application on VPS server. So, let's proceed:

## Installation

First check if the Python3 and pip is installed or not
```sh
python3 --version
pip --version
```
If not then install both python3 and pip
```sh
sudo apt install python3
sudo apt install python3-pip
sudo ln -s /usr/bin/pip3 /usr/bin/pip
```
Now Install virtualenv using pip
```sh
python3 -m pip install virtualenv
```
Now Install nginx
```sh
sudo apt update
sudo apt install nginx
```

> Note: `Installation` is required for a single time.


## Main Procress

Now I am going to upload two Django project in two different folder. 

```sh
/home/django/project-1/
/home/django/project-2/
```

At first, let us start with project-1:
First, navigate to the directory where the projects are
```sh
cd /home/django/
ls
```
If you find the directory of the Django project, then make a virtualenv for that project
```sh
python3 -m virtualenv project-1_venv
```
Activate the virtualenv and install the required packages
```sh
source project-1_venv/bin/activate
pip install -r project-1/requirements.txt
```
To check the installed packages
```sh
pip freeze
```
If you find that gunicorn is not installed then install it
```sh
pip install gunicorn
```
Navigate to the project directory and do migration, collectstatic and make a superuser to access the admin
```sh
cd project-1
python manage.py makemigrations
python manage.py migrate
python manage.py collectstatic
python manage.py createsuperuser
```
> Note: Make sure that in the project's `settings.py` file, `DEBUG = False` and `ALLOWED_HOSTS=['domain_name_or_ip_address']` are set.

Now make logs directory in the project folder and give them permission
```sh
sudo mkdir -p /home/django/project-1/logs
sudo chown -R root:root /home/django/project-1/logs
sudo chmod -R 775 /home/django/project-1/logs
```
> Note: `root:root` means the user is root and group is also root. You can navigate your user name by typing `whoami`.
Also you can check your other directory and file permission access by typing `ls -la`. If you want to change the user and group to root:root then type `chown -R root:root /home/django` `chmod -R 700 /home/django`.

Now, Make a `.socket` file
```sh
sudo nano /etc/systemd/system/project-1.socket
```
and type,
```sh
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/project-1.sock

[Install]
WantedBy=sockets.target
```
save it and then make a `.service` file
```sh
sudo nano /etc/systemd/system/project-1.service
```
and type,
```sh
[Unit]
Description=gunicorn daemon
Requires=project-1.socket
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/home/django/project-1
ExecStart=/home/django/project-1_venv/bin/gunicorn \
          --access-logfile /home/django/project-1/logs/gunicorn-access.log \
          --error-logfile /home/django/project-1/logs/gunicorn-error.log \
          --log-level debug \
          --capture-output \
          --workers 3 \
          --bind unix:/run/project-1.sock \
          MyProject.wsgi:application

[Install]
WantedBy=multi-user.target
```
> Note: MyProject is the directory of project where `settings.py` is exists.

Now start both the `.socket` and `.service` file
```sh
sudo systemctl start project-1.socket
sudo systemctl enable project-1.socket

sudo systemctl start project-1.service
sudo systemctl enable project-1.service
```

Check if it is running or not
```sh
sudo systemctl status project-1.service
```

After that to make the nginx file, type
```sh
sudo nano /etc/nginx/sites-available/project-1
```
and type
```sh
server {
    listen 80;
    server_name domain_name_or_ip_address;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        alias /home/django/project-1/staticfiles/;
    }
    location /media/ {
        alias /home/django/project-1/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/project-1.sock;
    }
}
```
save it and type
```sh
sudo ln -s /etc/nginx/sites-available/project-1 /etc/nginx/sites-enabled
```
Check if the syntax is ok or not
```sh
sudo nginx -t
```
Restart the nginx
```sh
sudo systemctl restart nginx
```

And then your project is live to the website.
Repeat the same procedure for other projects.

### Some Necessary Commands
To check the nginx log, type
```sh
sudo tail -F /var/log/nginx/error.log
```
If nginx is not restarting, type 
```sh
sudo lsof -i :80
sudo lsof -i :443
```
If you find httpd service is running, disable and kill that
```sh
sudo fuser -k 80/tcp
sudo fuser -k 443/tcp
sudo service httpd stop
sudo systemctl disable httpd
sudo killall httpd
```

To get `ssl` certificate for the domain, install certbot if it is not installed before
```sh
sudo apt install certbot python3-certbot-nginx
```
then 
```sh
sudo certbot --nginx -d your_domain -d www.your_domain
```

If you made any changes in the `.socket` or `.service`, then first type
```sh
sudo systemctl daemon-reload
```
and then restart `.socket` and `.service`
```sh
sudo systemctl restart project-1.socket
sudo systemctl restart project-1.service
```

If you made any changes in the project directory, then type
```sh
sudo systemctl restart project-1.service
sudo systemctl restart nginx
```



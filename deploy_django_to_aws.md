## 1. create instance and key-pair on AWS

  login AWS, EC2 Dashboard
 
  launch instance -->
  
  select Ubuntu Server 16.04 LTS (HVM), SSD Volume Type - ami-da05a4a0 -->
  
  select General purpose t2.micro (free tier eligible)  -->
  
  click review and launch -->
  
  after enter the instance, click Edit security groups
  
  select Type as SSH, Source as my IP --> add Rule   
  
  Type as HTTP, Source as anywhere
  
  click review and launch
  
  click launch
  
  under select an existing key pair or create a new key pair select: create a new key pair and name it as Django-app and download key-pair and launch instance, once the instance state changed from initializing to running, select this instance and then click connect.

## 2. connect local computer to the instance, downlaod jango app and install virtual environment and python packages  
### 2.1 copy the key to Desktop and connect to instance 
  ```
  cd downloads/
  
  mv Django-app.pem.txt ~/Desktop/
  
  chmod 400 Django-app.pem.txt
  ssh -i "Django-app.pem.txt" ubuntu@ec2-54-174-99-110.compute-1.amazonaws.com
  ```
select yes to connect to AWS

### 2.2 update apt-get and install python pip3
```
sudo apt-get update
sudo apt-get install python3-pip
sudo apt-get install python3-dev nginx git

sudo apt-get update
sudo pip3 install virtualenv
```
### 2.3 clone the django app from github account
```
git clone https://github.com/weifengshe/django-simple-example.git
```
### 2.4 move the learning_users to the parent directory and remove django-simple-example folder
```
cd django-simple-example/
mv requirements.txt learning_users/
mv * ../
rm -rf django-simple-example/
```
### 2.5 create virtual environment  and install required python packages
```
cd learning_users/
virtualenv -p /usr/bin/python3.5 venv
source venv/bin/activate
pip3 install -r requirements.txt
pip3 install django bcrypt django-extensions
pip3 install gunicorn
```
### 2.6 add the host ip address in settings.py
```
cd learning_users/
sudo vim settings.py
```
insert ip into ALLOWED_HOSTS  
```
ALLOWED_HOSTS = ['54.174.99.110'] # this ip is from my instance 
```
 add this line below to the bottom of the file

```
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```
comment this following line:
```
# STATICFILES_DIRS = [STATIC_DIR,]  
cd ..
python manage.py collectstatic
```

## 3. Set up server and load balancer


### 3.1 Server and load balancer setting  
#### gunicorn: a Python WSGI HTTP Server for UNIX.####
```
gunicorn --bind 0.0.0.0:8000 learning_users.wsgi:application
```
stop by crl+c

### 3.2 edit gunicorm.service
```
sudo vim /etc/systemd/system/gunicorn.service
# add the following code inside it
[Unit]
Description=gunicorn daemon
After=network.target
[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/learning_users
ExecStart=/home/ubuntu/learning_users/venv/bin/gunicorn --workers 3 --bind unix:/home/ubuntu/learning_users/learning_users.sock learning_users.wsgi:application
[Install]
WantedBy=multi-user.target
```
ESC:wq

```
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```
### 3.3 edit gunicorm.service
```
sudo vim /etc/nginx/sites-available/learning_users
# add the following code inside it
server {
  listen 80;
  server_name 54.174.99.110;
  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
      root /home/ubuntu/learning_users;
  }
  location / {
      include proxy_params;
      proxy_pass http://unix:/home/ubuntu/learning_users/learning_users.sock;
  }
}
```

```
sudo ln -s /etc/nginx/sites-available/learning_users /etc/nginx/sites-enabled
sudo nginx -t
sudo rm /etc/nginx/sites-enabled/default
sudo service nginx restart
```
### start app at 54.174.99.110

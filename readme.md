### Introduction

In this guide, we're going to be building a very simple Flask application that accepts a query string in the URL, renders a template and displays the query string keys & values in a table.

We're going to be deploying the app on a Google Cloud virtual machine using Ubuntu 18.04.

You can find the small amount of source code at the Github repo <a href="https://github.com/Julian-Nash/simple-flask-demo">Here</a> if you'd like to follow along. 

The puspose of this guide is to cover the setup of a VM and a basic introduction to deploying a Flask application with Nginx & uwsgi.

A few things to note about this guide:

- We won't be using a domain name
- We won't be creating certificates/serving HTTPS requests
- We'll be using Github as a remote repository

### How to follow this guide

There's a few ways to follow along:

- Clone this repo to your local machine and set up a new remote repository
- Copy the source and create the files/directories yourself on your local machine

Either way, you'll need to push your code to a remote repo as we'll be pulling the code into the virtual macine.

### Dependencies

- Flask
- uwsgi

### Running the application locally

- Create a new virtual environment with `python -m venv <name_of_your_environment>`
- Clone this repo or create the project files individually
- Activate the virtual environment
- Install the dependencies with `pip install -r requirments.txt` or with `pip install flask uwsgi`

If you're copying the source code and creating the files/directories yourself, be sure to generate a `requirements.txt` file by running the following command from the app parent directory:

```sh
pip freeze > requirements.txt
```

### Running the application with the Flask development server

In this example, the entrypoint to our application is `run.py`. 

Enter the following commands in the same directory as `run.py` to run the app with the Flask development server:

```sh
export FLASK_APP=run.py
export FLASK_ENV=development
flask run
```

Access the app in your browser at `127.0.0.1:5000`.

### Running the appliaction with uwsgi locally

To run the application locally with `uwsgi`, run the `uwsgi` command followed by the name of the development `ini` file:

```sh
uwsgi dev.ini
```

Access the app in your browser at `127.0.0.1:9090`.

### Server setup

In this example, we'll be deploying our application to a virtual machine on Google cloud platform.

Create a Google cloud account and/or sign into the console.

### Creating a free VM

- Create a new project
- Click the menu icon in the top left of the console
- Select Compute engine > VM instances
- Wait for Compute engine to get ready

- Click create
- Name the instance
- In `Machine type`, select micro (1 shared vCPU) - It's free!
- In `Boot disk`, select Ubuntu 18.04 LTS and click select
- In the `Firewall` section, tick both Allow HTTP traffic and Allow HTTPS traffic
- Leave everything as is and click create
- Wait for the instance to become ready

You'll see your External IP address, make a note of it for later!

### Adding a network tag

Now that the instance is ready, we need to add a network tag to enable us to test the application using `uwsgi` (Optional)

- Click on the VM instance
- Click `EDIT` at the top of the page
- In the `Network tags` section, add `flask` (You may need to add a comma after it)
- Scroll down and hit `Save`

### Adding new a firewall rule

We're going to use port 9090 to test the application with `uwsgi`. But first, we need to add a new firewall rule. (Optional)

- Select the menu in the Google cloud console
- Click VPC network > Firewall rules
- Click `Create firewall rule` at the top of the page
- Name it `uwsgi-testing`
- Give it a description of `uwsgi testing on port 9090`
- In `Targets`, select `Specified target tags`
- In `Target tags`, enter `flask`
- In `Source filter`, select `IP Ranges`
- In `Source IP ranges`, enter `0.0.0.0/0`
- In `Protocols and ports`, select `Specified protocols and ports`
- Select `TCP` and enter `9090`
- Click `Create`

To make sure the new firewall rule has been applied:

- Navigate back to the VM instance `Menu/Compute engine/VM instances`
- Click on the menu icon next to your VM instance and select `View network details`

In the firewall rules and routes details section, you should see `uwsgi-testing`.

We're going to come back and disable this rule after we've tested the application with uwsgi!

### Connecting to the VM

We're going to use the Google cloud shell provided to connect to our VM.

- Click the `SSH` button under the `Connect` section to launch a terminal
- A new shell should be spawned with your username@instance in the prompt

### Update the machine

Update the system packages:

```sh
sudo apt update -y;sudo apt upgrade -y
```

### Installing Python3.7

We're going to use Python3.7.2 and use `pyenv` to manage our Python installations.

Clone the `pyenv` repo (It will clone into your user home directory by default):

```sh
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

For `pyenv` to work, you'll need to add a few lines to your `.bashrc` file.

Run the following commands to update your `.bashrc` and reload the shell:

```sh
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
exec "$SHELL"
```

Install the required Python build dependencies:

```sh
sudo apt-get update; sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
```

Once the dependencies have been installed, we can install Python3.7.2:

```sh
pyenv install 3.7.2
```

You should see (This may take some time on the micro instance!):

```sh
Downloading Python-3.7.2.tar.xz...
-> https://www.python.org/ftp/python/3.7.2/Python-3.7.2.tar.xz
Installing Python-3.7.2...
```

Once installed, run the following command to check Python3.7.2 has been installed:

```sh
pyenv versions
```

You should see `3.7.2`.

Now set the system Python as 3.7.2:

```sh
pyenv global 3.7.2
```

Start a python interpreter with the `python` command to double check the right version is being called. You should see:

```pycon
Python 3.7.2 (default, Mar 20 2019, 23:12:56) 
[GCC 7.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

### Cloning the app

We'll need a method to get the application code from your local machine to the VM.

Assuming you've cloned this repo to your local machine and set up a new repote origin for yourself, or just copied the code and setup a new repo, we're going to clone into the app on the virtual machine.

Move into the home directory:

```sh
cd ~/
```

Clone the repo, being sure to replace the URL with the URL of YOUR repo!:

```sh
git clone https://github.com/Julian-Nash/flask-demo.git
```

**IMPORTANT**

If you've cloned this repo, rename the `simple-flask-demo` parent directory to `app`. Otherwise, just make sure the application parent directory is named `app`.

If you need to rename `simple-flask-demo` on the virtual machine, you can do so with: 

```sh
mv simple-flask-demo/ app
```

The file/directory structure should look like this (where the parent `app` directory is located in your user home directory):

```sh
app
├── app
│   ├── __init__.py
│   ├── templates
│   │   └── public
│   │       └── index.html
│   └── views.py
├── app.ini
├── config.py
├── dev.ini
├── readme.md
├── requirements.txt
└── run.py
```

### Installing the dependencies

We need to create a new virtual environment and install the required packages.

Move into the parent `app` directory:

```sh
cd app
```

Running the `ls` command should return:

```sh
app  app.ini  config.py  dev.ini  readme.md  requirements.txt  run.py
```

Create a new virtual environment. We're going to call ours `env` (You should too!):

```sh
python -m venv env
```

Activate it:

```sh
source env/bin/activate
```

Upgrade pip:

```sh
pip install --upgrade pip
```

Install the Python dependencies (This may take a few minutes on a micro instance):

```sh
pip install -r requirements.txt
```

### Testing

We can quickly test our application using uwsgi.

First, we need to add a firewall rule using `ufw`:

```sh
sudo ufw allow 9090
```

Now, make sure `ufw` is enabled:

```sh
sudo ufw enable
```

Run the following to make sure `ufw` is enabled and port 9090 is exposed:

```sh
sudo ufw status
```

You should see:

```sh
Status: active
To                         Action      From
--                         ------      ----
9090                       ALLOW       Anywhere                  
9090 (v6)                  ALLOW       Anywhere (v6) 
```

### Running with uwsgi

Assuming the following:

- Your virtual environment is active
- You've created a new firewall rule to allow `9090` in the Google Cloud console
- You've enabled `ufw` and added `9090` as a rule on the VM

Make sure you're in the same directory as `dev.ini` and run the application with the following:

```sh
uwsgi dev.ini
```

You should see some output from uwsgi in the terminal to let you know uwsgi has started and is running.

open up a new browser window and head to your virtual machines IP address followed by `:9090`, for example:

```sh
http://35.237.110.230:9090
```

You should see the application running! Feel free to send a query string in the URL to have it parsed and returned in the table.

```sh
http://<your_ip_address>/?foo=hello&bar=world&flask=awesome
```

We're going to be using Nginx as a reverse proxy to handle HTTP requests, so once you've had some fun with the application, stop uwsgi with `Ctrl + c`.

### Disable the development port

We used `ufw` to enable traffic on port `9090` but now we need to delete it:

```sh
sudo ufw delete allow 9090
```

Run `sudo ufw status` to confirm the rule has been deleted. You should see:

```sh
Status: active
```

### Disabling the firewall rule in GCP

We should remove the firewall rule we created for testing in the Google cloud console.

- Navigate to Menu > VPC networking > Firewall rules
- Click `uwsgi-testing` from the list of rules
- Click `Delete` at the top of the page to remove the rule

If you head back to your VM instance and select `View network details` from the dropdown menu, you'll see the firewall rule has been removed.

### Installing Nginx

We're going to use Nginx to handle incoming HTTP requests to our application, so we need to install and configure it.

In stall Nginx with the following command:

```sh
sudo apt install nginx
```

### Adjusting the UFW firewall

Just like how we enabled port `9090` for testing our app with `uwsgi`, we need to enable a few ports to enable Nginx.

`ufw` will see it as an available application if it's installed. We can chack this by running:

```sh
sudo ufw app list
```

You'll see something similar to this:

```sh
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

We're only going to be serving our application over HTTP on port 80, so we need to enable it with the following:

```sh
sudo ufw allow 'Nginx HTTP'
```

This will allow HTTP traffic on port `80`, the default HTTP port.

We can check the rule has been applied with:

```sh
sudo ufw status
```

You should see:

```sh
Status: active
To                         Action      From
--                         ------      ----
Nginx HTTP                 ALLOW       Anywhere                  
Nginx HTTP (v6)            ALLOW       Anywhere (v6) 
```

### Checking Nginx

We can see Nginx is running by heading the the IP address of the virtual machine in a new browser window.

```sh
http://<your_ip_address>
```

You should be greeted with a default `Welcome to nginx!` page.

We can also check with `systemd` that Nginx is running.

`systemd` is a software suite that manages services and processes and will start Nginx when your server boots:

```sh
systemctl status nginx
```

You should see:

```sh
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-03-21 12:09:59 UTC; 6min ago
     Docs: man:nginx(8)
  Process: 5761 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 5749 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 5764 (nginx)
    Tasks: 2 (limit: 667)
   CGroup: /system.slice/nginx.service
           ├─5764 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           └─5767 nginx: worker process
```

There's still a few more steps before we can access our application:

- Configure Nginx to reverse proxy requests to uwsgi
- Create a `systemd` unit file to automatically start `uwsgi` and serve our app

We'll start by creating the `systemd` unit file.

### Systemd unit file

We're going to use `nano` to create a `.service` unit file. We'll call ours `app.service`.

**IMPORTANT**

This guide assumes you've used the same directory and file names that we've used throughout this tutorial.

You'll also need to replace `<yourusername>` with your actual username! You can see your username in your terminal prompt I.e `your-username@your-instance`

Open `nano` with the following:

```sh
sudo nano /etc/systemd/system/app.service
```

Add the following:

```sh
[Unit]
Description=A simple Flask uWSGI application
After=network.target

[Service]
User=<yourusername>
Group=www-data
WorkingDirectory=/home/<yourusername>/app
Environment="PATH=/home/<yourusername>/app/env/bin"
ExecStart=/home/<yourusername>/app/env/bin/uwsgi --ini app.ini

[Install]
WantedBy=multi-user.target
```

Save and close the file with `Ctrl + c`, followed by `y` then `Enter`.

Start the process:

```sh
sudo systemctl start app
```

Enable the process:

```sh
sudo systemctl enable app
```

Check the process status:

```sh
sudo systemctl status app
```

You should see:

```sh
● app.service - A simple Flask uWSGI application
   Loaded: loaded (/etc/systemd/system/app.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-03-21 14:48:12 UTC; 9min ago
 Main PID: 7580 (uwsgi)
    Tasks: 5 (limit: 667)
   CGroup: /system.slice/app.service
           ├─7580 /home/julianjamesnash/app/env/bin/uwsgi --ini app.ini
           ├─7592 /home/julianjamesnash/app/env/bin/uwsgi --ini app.ini
           └─7595 /home/julianjamesnash/app/env/bin/uwsgi --ini app.ini
```

Now we can move on to the final step, configuring Nginx!

### Configuring Nginx

We need to create a new server block in Nginx's `sites-available`. We'll use `nano` again to create a new file called `app`:

```sh
sudo nano /etc/nginx/sites-available/app
```

**IMPORTANT**

Just as with the `systemd` unit file, you'll need to replace `<username>` with your username and `<your_ip_address>` with the IP address of your virtual machine!

Add the following:

```sh
server {
    listen 80;
    server_name <your_ip_address>;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/<username>/app/app.sock;
    }
}
```

If you'd like to use your own domain name, replace `<your_ip_address>` with:

```sh
server_name example.com www.example.com;
```

You'll need update your domain registrar to point the domain to the server IP address if you want to use a custom domain, which we're not going to cover in this guide.

We need to link the server block we've just created in `sites-available` to `sites-enabled`:

```sh
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled
```

We can check Nginx for syntax errors with the following:

```sh
sudo nginx -t
```

You should see:

```sh
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

We can now restart the Nginx service:

```sh
sudo systemctl restart nginx
```

### Testing the application

Assuming you've not had any syntax errors or used different directory/filenames. Go to your IP address in a browser and you should see the application in action.

Try sending a query string in the URL such as:

```sh
/?foo=hello&bar=world&flask=awesome
```

You should see the query string arguments displayed in the table!

### Updating your app

In this scenario, the best way to make changes to your application:

- Make changes and test locally
- Push the changes to your remote Github repo
- Pull the changes from your virtual machine

To pull any changes to you've made to your application, make sure you're in the `app` parent directory:

```sh
cd ~/app
```

Pull the repo with the following command:

```sh
git pull
```

Every time you pull any changes from your remote repo, you'll need to restart the `app` service with:

```sh
sudo systemctl restart app
```

If you make any changes to the Nginx `sites-enabled` file, you'll need to restart Nginx with:

```sh
sudo systemctl restart nginx
```

### Wrapping up

This was just a quick guide to deploying a Flask app to a virtual machine using Nginx & uWSGI and as you can see, it's relitively simple.

We used Google Cloud but of course, you could achieve the same result using any other provider such as AWS, Linode, Digital Ocean etc. If you do decide to use another cloud platform, you may not have to bother configuring custom firewall rules, it's really platform dependent.

By modern standards, deploying an application this way may seem slow, especially when compared to using Docker or a hosted service like Google App Engine, however I hope it demonstrates that it's really not that difficult!
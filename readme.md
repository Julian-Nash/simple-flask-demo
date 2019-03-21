### Introduction

In this guide, we're going to be building a very simple application that doesn't really do anything special, it just accepts a query string, renders a template and displays the keys & values in a table.

The puspose of this guide is to cover the setup of the virtual machine.

### Dependencies

- Flask
- uwsgi

### Running the application locally

- Clone this repo
- Create a new virtual environment with `python -m venv <name_of_your_environment>`
- Activate the virtual environment
- Install the dependencies with `pip install -r requirments.txt`

### Running the application with the Flask development server

Run the following commands in the same directory as `run.py` to run the app with the Flask development server:

```sh
export FLASK_APP=run.py
export FLASK_ENV=development
flask run
```

Access the app in your browser at `127.0.0.1:5000`.

### Running the appliaction with uwsgi locally

```sh
uwsgi dev.ini
```

Access the app in your browser at `127.0.0.1:9090`.

### Server setup

In this example, we'll be deploying our application to a virtual machine on Google cloud platform.

Create a Google cloud account and/or sign into the console.

### Creating a free VM

- Create a new project
- Click the top left hamburger menu
- Select compute engine > VM instances
- Wait for Compute engine to get ready

- Click create
- Name the instance
- In `Machine type`, select micro (1 shared vCPU) - It's free!
- In `Boot disk`, select Ubuntu 18.04 LTS and click select
- In the `Firewall` section, tick both Allow HTTP traffic and Allow HTTPS traffic
- Leave everything as is and click create
- Wait for the instance to become ready

### Connecting to the VM

We're going to use the shell provided to connect to our VM.

- Click the `SSH` button under the `Connect` section to launch a terminal
- A new shell should be spawned with your username@instance in the prompt

### Update the machine

```sh
sudo apt update -y;sudo apt upgrade -y
```

### Installing Python3.7

We're going to use Python3.7.2 and use `pyenv` to manage our Python installations.

Clone the repo:

```sh
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

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

Once installed, run the following command to check 3.7.2 has been installed:

```sh
pyenv versions
```

You should see `3.7.2`.

Set the system Python as 3.7.2:

```sh
pyenv global 3.7.2
```

Start a python instance with the `python` command to double check. You should see:

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

Create a new directory called `app` and move into it:

```sh
mkdir app
cd app
```

Clone the repo:

```sh
git clone https://github.com/Julian-Nash/flask-demo.git
```

Rename the directory to `app`:

```sh
mv simple-flask-demo/ app
```

### Installing the dependencies

We need to create a new virtual environment and install the required packages.

Move into the `app` directory:

```sh
cd app
```

Running the `ls` command should return:

```sh
app  app.ini  config.py  dev.ini  readme.md  requirements.txt  run.py
```

Create a new virtual environment. We're going to call ours `env` (You probably should too!):

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

Before we go any further with our application, we need to install and configure Nginx.

### Installing Nginx


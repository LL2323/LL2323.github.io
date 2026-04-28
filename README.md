# Build Break Repeat - Infrastructure Foundation

A tool for deploying and managing a CTF competition environment on a single Linux host using Docker.

## Requirements
- Linux (CentOS/Ubuntu recommended)
- Docker
- Terraform
- Git
- Curl
- Wget
- Python 3

Installing the basics:

```bash
sudo apt update
sudo apt install git -y
sudo apt install curl -y
```

## Setup

```bash
git clone https://github.com/Build-Break-Repeat/CTF_Framework.git
cd CTF_Framework
bash init.sh
```

## Usage

```bash
# Enter Sudo for Administrator Privilege
sudo su

# Deploy CTFCTL and Follow on Screen Prompts
./ctfctl deploy
```

## Additional Usage Commands

```bash
# Check running containers and their URLs
./ctfctl status

# Tear everything down
./ctfctl destroy

# Destroy then redeploy in one command
./ctfctl rebuild
```

## Configuration

Challenges and event settings are managed through `config.json`. Use the CLI to edit instead of touching the file directly:

```bash
# Add, edit, remove, or list challenges
./ctfctl challenge add
./ctfctl challenge list
./ctfctl challenge edit

# Set event name, team count, admin credentials
./ctfctl event configure
```

## Flags

Flags are generated per team and injected automatically during `deploy`. To run manually:

```bash
./ctfctl flags generate
./ctfctl flags inject
```

## Challenge Ports

See [Documentation.txt](Documentation.txt) for the full list of challenges and their ports.

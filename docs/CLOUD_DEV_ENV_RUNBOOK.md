# Linode Remote Development Environment Runbook

## 1. Create a Linode Instance

1. Log in to your Linode account (or create one if you haven't already).
2. Click "Create Linode" in the dashboard.
3. Choose a distribution: Select Ubuntu 22.04 LTS.
4. Select a region close to your location for better latency.
5. Choose a Linode plan:
   - For development, a Shared CPU instance with 4GB RAM and 2 vCPUs should be sufficient.
   - Example: Linode 4GB plan
6. Create a secure root password.
7. Add an SSH key for secure access (recommended).
8. Click "Create Linode" and wait for it to provision.

## 2. Initial Server Setup

1. Once your Linode is running, copy its IP address from the Linode dashboard.
2. Open a terminal on your local machine and SSH into your Linode:
   ```
   ssh root@your_linode_ip
   ```
3. Update the system:
   ```
   apt update && apt upgrade -y
   ```
4. Create a new user with sudo privileges:
   ```
   adduser devuser
   usermod -aG sudo devuser
   ```
5. Set up a firewall:
   ```
   ufw allow OpenSSH
   ufw enable
   ```

## 3. Install Docker

1. Install Docker dependencies:
   ```
   sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
   ```
2. Add Docker's official GPG key:
   ```
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   ```
3. Add Docker repository:
   ```
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```
4. Update apt and install Docker:
   ```
   sudo apt update
   sudo apt install docker-ce docker-ce-cli containerd.io -y
   ```
5. Add your user to the Docker group:
   ```
   sudo usermod -aG docker devuser
   ```
6. Log out and log back in for the group changes to take effect:
   ```
   exit
   ssh devuser@your_linode_ip
   ```

## 4. Install and Configure code-server

1. Create a directory for code-server:
   ```
   mkdir -p ~/.config/code-server
   ```
2. Create a config file:
   ```
   nano ~/.config/code-server/config.yaml
   ```
3. Add the following content (replace `your_password` with a secure password):
   ```yaml
   bind-addr: 0.0.0.0:8080
   auth: password
   password: your_password
   cert: false
   ```
4. Save and exit (Ctrl+X, then Y, then Enter).
5. Run code-server in a Docker container:
   ```
   docker run -d --name code-server -p 8080:8080 \
     -v "$HOME/.config:/home/coder/.config" \
     -v "$HOME/project:/home/coder/project" \
     -u "$(id -u):$(id -g)" \
     -e "DOCKER_USER=$USER" \
     codercom/code-server:latest
   ```

## 5. Set Up Nginx as a Reverse Proxy (for HTTPS)

1. Install Nginx:
   ```
   sudo apt install nginx -y
   ```
2. Install Certbot for SSL:
   ```
   sudo apt install certbot python3-certbot-nginx -y
   ```
3. Create an Nginx configuration file:
   ```
   sudo nano /etc/nginx/sites-available/code-server
   ```
4. Add the following content (replace `your_domain` with your actual domain):
   ```nginx
   server {
       listen 80;
       server_name your_domain;

       location / {
           proxy_pass http://localhost:8080;
           proxy_set_header Host $host;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection upgrade;
           proxy_set_header Accept-Encoding gzip;
       }
   }
   ```
5. Enable the site:
   ```
   sudo ln -s /etc/nginx/sites-available/code-server /etc/nginx/sites-enabled/
   ```
6. Test Nginx configuration:
   ```
   sudo nginx -t
   ```
7. If the test is successful, reload Nginx:
   ```
   sudo systemctl reload nginx
   ```
8. Obtain an SSL certificate:
   ```
   sudo certbot --nginx -d your_domain
   ```
   Follow the prompts to complete the SSL setup.

## 6. Access Your Development Environment

1. Open a web browser and navigate to `https://your_domain`.
2. Log in using the password you set in the code-server config file.

## 7. Install Development Tools

Once you're in the code-server environment:

1. Open a terminal in code-server.
2. Install Node.js and npm:
   ```
   curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
   sudo apt install nodejs -y
   ```
3. Install Python and pip:
   ```
   sudo apt install python3 python3-pip -y
   ```
4. Install Flask:
   ```
   pip3 install flask
   ```
5. Install PostgreSQL client:
   ```
   sudo apt install postgresql-client -y
   ```

## 8. Clone Your Project Repository

1. In the code-server terminal:
   ```
   cd ~/project
   git clone https://github.com/your-username/threadster.git
   cd threadster
   ```
2. Install project dependencies:
   ```
   npm install  # For frontend
   pip3 install -r requirements.txt  # For backend, if you have a requirements file
   ```

## 9. Set Up PostgreSQL Database

1. Create a PostgreSQL container:
   ```
   docker run -d --name postgres -e POSTGRES_PASSWORD=your_db_password -p 5432:5432 postgres:13
   ```
2. Create a database for your project:
   ```
   docker exec -it postgres psql -U postgres -c "CREATE DATABASE threadster;"
   ```

## 10. Configure Environment Variables

1. Create a `.env` file in your project directory:
   ```
   nano ~/project/threadster/.env
   ```
2. Add necessary environment variables:
   ```
   DATABASE_URL=postgresql://postgres:your_db_password@localhost:5432/threadster
   FLASK_ENV=development
   FLASK_APP=app.py
   ```

## 11. Start Your Development Server

1. For the Flask backend:
   ```
   flask run --host=0.0.0.0
   ```
2. For the React frontend (in a new terminal):
   ```
   npm start
   ```

You now have a fully functional remote development environment for Threadster!

## Additional Notes

- Remember to commit and push your changes regularly to your Git repository.
- To stop the environment, you can stop the Docker containers and Nginx when not in use to save resources.
- Regularly update your system and Docker images for security:
  ```
  sudo apt update && sudo apt upgrade -y
  docker pull codercom/code-server:latest
  ```
- Monitor your Linode's resource usage and upgrade if necessary.
# PostgreSQL

## PostgreSQL Server on VM Ubuntu (Virtualbox) on Mac

- Download Virtualbox from https://www.virtualbox.org/wiki/Downloads
- Download Ubuntu from https://ubuntu.com/download/desktop (Look for the ARM document)
- Open Virtualbox
  - Create a new Enviroment 
  - Choose the ARM File 
  - Choose all the settings 
  - Remember the Password 'changeme'
  - Install Ubuntu -> Open the Terminal
- 'sudo apt update' Update the Package list
- 'sudo apt install postgresql postgresql-contrib' install Postgresql
- 'sudo systemctl status postgresql' Verify Installation and Service Status
- 'sudo -i -u postgres' Switch to the postgres User
- Check if the Server is running
  - 'sudo systemctl status postgresql'
  - 'sudo systemctl start postgresql'
  - 'psql -U postgres'
- 'psql' Access the PostgreSQL Prompt
- 'CREATE ROLE myuser WITH LOGIN PASSWORD 'mypassword';' Create a New Database User (Role)
- ' ALTER ROLE myuser CREATEDB;' Give the user the ability to create Databases
- 'CREATE DATABASE mydatabase OWNER myuser;' Create a New Database
- '\q' Exit psql and the postgres user session
- Edit pg_hba.conf 'sudo nano /etc/postgresql/14/main/pg_hba.conf' - 'host    all             all             0.0.0.0/0               md5'
- Configure the Virtualbox Ports
- 
 

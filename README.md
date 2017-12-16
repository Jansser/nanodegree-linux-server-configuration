# Linux Server Configuration
Description

## Server details

**IP Address**
18.216.231.41

**SSH port**
2200

**URL to web application**


## Summary

### Software installed
Pla

#### Secure your server
* Update all currently installed packages.
    `sudo apt-get update`
    `sudo apt-get upgrade`

* Change the SSH port from `22` to `2200`.
  * Configure the Lightsail firewall to allow port `TCP 2200` in the Networking tab.
  * Change the line "Port 22" to "Port 2200".
    `sudo vi /etc/ssh/sshd_config`
  * Restart ssh service.
    `sudo service ssh restart`

* Configure Firewall
    * Set the firewall to block any connections coming in, allow everything outgoing.
    Then allow our `2200` SSH port, `80` port to our web server and `123` NTP port and enable the firewall.
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 123/udp
    sudo ufw enable
    ```

### Give grader access.
  * Create a new user account named grader.
    `sudo adduser grader`

  * Give grader the permission to sudo.
    * Create grader file on sudoers.d
      `sudo touch /etc/sudoers.d/grader`

    * Edit the file to add the permissions.
      `sudo nano /etc/sudoers.d/grader`
      Add this line `grader ALL=(ALL) NOPASSWD:ALL`

  * Create an SSH key pair for grader using the ssh-keygen tool.
    * Run `ssh-keygen` on the local machine to create the key pair.
      create `grader_key` file on `.ssh` directory.

    * Place the public key on the server
      * Create directory `.ssh`
      * Create `authorized_keys` file
        `touch .ssh/authorized_keys`
      * Copy the public key from the local machine and paste in the `authorized_keys` file
        `sudo nano .ssh/authorized_keys`

      * Set permissions of the `.ssh` directory and the `authorized_keys` file
        `sudo chmod 700 .ssh`
        `sudo chmod 644 .ssh/authorized_keys`

    * Disable password authentication
      `sudo nano /etc/ssh/sshd_config`
      Change PasswordAuthentication to no

      * Restart ssh servie
      `sudo service ssh restart`

      Now we can connect with the grader user
      `sudo ssh grader@18.216.231.41 -p 2200 -i .ssh/grader_key`

### Prepare to deploy your project.
  * Configure the local timezone to UTC
    `sudo dpkg-reconfigure tzdata` select 'None of the above' and then UTC.

  * Install and configure Apache
    `sudo apt-get install apache2`

  * Install and configure Apache to serve a Python mod_wsgi application.
    `sudo apt-get install libapache2-mod-wsgi-py3`

  * Install git
    `sudo apt-get install git`

## Resources
https://br.godaddy.com/help/como-alterar-a-porta-ssh-para-seu-servidor-linux-7306

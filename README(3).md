# Deploy MongoDB Community on an Ubuntu EC2 instance

This guide installs MongoDB Community Edition on an Ubuntu EC2 virtual machine. It uses MongoDB's official APT repository and a Bash provisioning script so the installation steps can be repeated without typing each command manually.

> **Example setup:** AWS EC2, Ubuntu Server 24.04 LTS, 64-bit architecture, MongoDB 9.0, default port `27017`.
>
> MongoDB listens only on the instance itself by default. Keep that setting unless you have a specific need for remote access. If you do need remote access, follow the security section below before opening the port.

## What you need

- An AWS account with permission to create and terminate EC2 instances.
- A Windows computer with PowerShell and OpenSSH (`ssh` and `scp`). Current Windows versions include OpenSSH. You can check with `ssh -V`.
- An EC2 key pair saved as a `.pem` file. Keep it private and do not commit it to GitHub.
- An Ubuntu Server 24.04 LTS EC2 instance. Choose a 64-bit architecture supported by MongoDB and use a key pair you can access.

## 1. Launch the EC2 instance

1. Sign in to the AWS Console and open **EC2** in the correct region.
2. Select **Launch instance** and give it a name, such as `mongodb-training`.
3. Under **Application and OS Images**, choose **Ubuntu Server 24.04 LTS**. Select a 64-bit x86 image for the simplest match to the example.
4. Choose an instance type suitable for a small learning exercise, such as `t3.micro` if available in your region and account.
5. Select or create a key pair. Download the `.pem` file and store it somewhere private, for example `C:\Users\YourName\.ssh\`.
6. In **Network settings**, create or select a security group. Allow **SSH / TCP 22** from **My IP** only.
7. Leave inbound port `27017` closed for now. MongoDB will first be tested locally on the EC2 machine.
8. Review the settings and select **Launch instance**.
9. Wait until the instance state is **Running** and both status checks have passed. Copy its **Public IPv4 address**.

## 2. Connect from Windows PowerShell

Open PowerShell and change to the folder containing your key and the downloaded project files. Replace the example path and address with your own values:

```powershell
cd $HOME\Downloads\mongodb-ec2
ssh -i "$HOME\.ssh\my-ec2-key.pem" ubuntu@203.0.113.10
```

The first time you connect, type `yes` if asked to trust the host. The prompt should change to the Ubuntu instance. For example:

```text
ubuntu@ip-10-0-1-25:~$
```

If SSH reports that the key is too open, restrict its permissions in PowerShell:

```powershell
icacls "$HOME\.ssh\my-ec2-key.pem" /inheritance:r
icacls "$HOME\.ssh\my-ec2-key.pem" /grant:r "$($env:USERNAME):(R)"
```

## 3. Copy the provisioning script to EC2

Open a **second PowerShell window** on your Windows computer (keep the SSH window open). From the folder containing `mongo_setup.sh`, upload it with `scp`:

```powershell
scp -i "$HOME\.ssh\my-ec2-key.pem" .\mongo_setup.sh ubuntu@203.0.113.10:~
```

Replace the key path and IP address. The file should now be in the Ubuntu user's home folder.

## 4. Run the provisioning script

In the SSH window connected to EC2, run:

```bash
chmod +x ~/mongo_setup.sh
~/mongo_setup.sh
```

The script detects the Ubuntu release, adds MongoDB's signed APT repository, installs MongoDB Community, starts the service, enables it to start after a reboot, and checks that it responds to a ping.

The script supports Ubuntu 22.04 LTS (`jammy`) and 24.04 LTS (`noble`) in this example. It stops with an explanation if it detects another release or CPU architecture.

## 5. Check MongoDB

Check that the service is active:

```bash
sudo systemctl status mongod --no-pager
```

The service should show `active (running)`. Press `q` if you opened the status pager without `--no-pager`.

Connect using the MongoDB Shell and run a quick check:

```bash
mongosh
```

At the `mongosh` prompt, enter:

```javascript
db.runCommand({ ping: 1 })
```

A successful response includes `ok: 1`. Exit the shell with:

```javascript
exit
```

Useful service commands:

```bash
sudo systemctl restart mongod
sudo systemctl stop mongod
sudo systemctl start mongod
```

## 6. Optional: allow a remote client to connect

Do this only if the task requires connecting to MongoDB from your Windows computer. MongoDB's default configuration listens on localhost, so it cannot be reached remotely. **Do not open port `27017` to everyone (`0.0.0.0/0`).** Publicly reachable databases are routinely scanned, and an unprotected database can be accessed or damaged by others.

### Safer choice: SSH tunnel

An SSH tunnel lets your computer connect through the existing SSH rule. It avoids adding an inbound MongoDB rule or making MongoDB listen on the public interface.

In a separate PowerShell window, run and leave this command open:

```powershell
ssh -i "$HOME\.ssh\my-ec2-key.pem" -L 27017:127.0.0.1:27017 ubuntu@203.0.113.10
```

Then connect your local MongoDB client to:

```text
mongodb://127.0.0.1:27017
```

### Assignment option: add a restricted inbound rule

If you have been specifically asked to add an inbound rule for port `27017`:

1. First enable MongoDB authentication. Connect locally with `mongosh` and create an administrator account:

   ```javascript
   use admin
   db.createUser({
     user: "mongoAdmin",
     pwd: passwordPrompt(),
     roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase", "dbAdminAnyDatabase" ]
   })
   ```

   Enter a strong, unique password when prompted. Do not put the password in this README, the script, or GitHub.

2. In the EC2 session, get the instance's private IPv4 address:

   ```bash
   hostname -I
   ```

   Use the private address shown by this command in the next step.

3. Edit the configuration:

   ```bash
   sudo nano /etc/mongod.conf
   ```

   Under the existing `net:` section, set `bindIp` to localhost and the EC2 **private** address. Preserve YAML indentation and do not add a second `net:` section. Example only:

   ```yaml
   net:
     port: 27017
     bindIp: 127.0.0.1,10.0.1.25
   ```

   Under `security:`, enable authorization. If no `security:` section exists, add one at the top level:

   ```yaml
   security:
     authorization: enabled
   ```

   Save in nano with **Ctrl+O**, press **Enter**, then exit with **Ctrl+X**.

4. Restart MongoDB and confirm it is running:

   ```bash
   sudo systemctl restart mongod
   sudo systemctl status mongod --no-pager
   ```

5. In the AWS Console, open **EC2 → Security Groups → select the instance's security group → Inbound rules → Edit inbound rules → Add rule**. Set:
   - Type: **Custom TCP**
   - Port range: **27017**
   - Source: **My IP** (or the specific trusted client IP in CIDR form, such as `198.51.100.24/32`)

   Save the rule. Do not select **Anywhere IPv4**.

6. Connect from a client using the instance's public IPv4 address, port `27017`, and the database user credentials. Use an encrypted connection for real or sensitive data; this learning setup does not configure TLS. The SSH tunnel above is preferred for a short training exercise.

If you no longer need remote access, remove the inbound rule and return `bindIp` to `127.0.0.1`, then restart `mongod`.

## 7. Finish the exercise and avoid ongoing charges

When finished, save any work you need. In the AWS Console, open **EC2 → Instances**, select the training instance, choose **Instance state → Terminate instance**, and confirm. Stopping an instance may still leave billable resources such as attached storage or reserved public IPv4 addresses; check and remove anything the exercise no longer needs.

## Troubleshooting

- **`Unable to locate package mongodb-org`:** Check the Ubuntu release with `lsb_release -cs`, then confirm it is `jammy` or `noble`. Check that the script completed the repository setup and rerun `sudo apt update`.
- **MongoDB did not start:** Inspect the service and log:

  ```bash
  sudo systemctl status mongod --no-pager
  sudo tail -n 50 /var/log/mongodb/mongod.log
  ```

- **SSH times out:** Confirm the instance is running, the public IPv4 address is current, and the security group allows SSH from your current public IP.
- **Remote MongoDB connection times out:** Check that the service is active, the `bindIp` includes the EC2 private address, and the security group source is your trusted public IP. Avoid opening the port to all sources.

## References

- [MongoDB Community installation instructions for Linux](https://www.mongodb.com/docs/manual/administration/install-community-linux/?linux-distro=ubuntu&linux-method=pkg)
- [MongoDB configuration options (`net.bindIp`)](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.bindIp)
- [MongoDB security checklist](https://www.mongodb.com/docs/manual/administration/security-checklist/)
- [AWS EC2 instance termination](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html)

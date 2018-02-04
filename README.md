# ADVANCE Cold-Hot wallet MasterNode setup guide

> This is a community contributed guide. Feel free to suggest improvements via Issues or opening Pull Requests. Thank you!

## **Cold** Wallet Setup(Part 1)

This is the wallet where the MasterNode collateral of 1000 ADVANCE will have to be transferred and stored. After the setup is complete, this wallet doesn't have to run 24/7 and will be the one receiving the rewards.

1. Open wallet on your desktop.

   Click Settings -> Options -> Wallet
   
   Check "Enable coin control features"
   
   Check "Show Masternodes Tab"
   
   ![Alt text](https://github.com/tech1e/cold-hot-advance/blob/master/EnableFeatures.png "Wallet Settings")
   
   Press **Ok** 
   
   Restart ADVANCE Wallet and wait until it syncs to ADVANCE network. Check status by hovering the mouse pointer over the right bottom icons.
   
   ![Alt text](https://github.com/tech1e/cold-hot-advance/blob/master/WalletRestartAndSync.png "Wallet Status")

2. Create a receiving address for the Masternode collateral funds.

   Go to File -> Receiving addresses...
   
   Click **New**, type in a label and press **Ok**.
   
   ![Alt text](https://github.com/tech1e/cold-hot-advance/blob/master/ReceivingAddressesNewAddress.png "Receiving Address")

3. Select the row of the newly added address and click **Copy** to store the destination address in the clipboard.
4. Send exactly 1000 ADVANCE to the address you just copied. Double check you've got the correct address before transferring the funds.
5. After sending, you can verify the balance in the Transactions tab. This can take a few minutes to be confirmed by the network.
6. Open the debug console of the wallet in order to type a few commands. 

   Go to `Tools` -> `Debug console`
   
7. Run the following command: `masternode genkey`

   ![Alt text](https://github.com/tech1e/cold-hot-advance/blob/master/GenKey.png "Masternode Key")

   You should see a long key that looks like:
   ```
   3HaYBVUCYjEMeeH1Y4sBGLALQZE1Yc1K64xiqgX37tGBDQL8Xg
   ```
   We will use this later on both cold and hot wallets.

8. Run `masternode outputs` command to retrieve the transaction ID of the collateral transfer.

   You should see an output that looks like this:
   ```
   [
      {
        "txhash" : "6782efab3a76fa557370ec3b9c13bf0d0df3d4df63adc018e1dd90e1c8da088e",
        "outputidx" : 1
      }
   ]
   ```

   Both `txhash` and `outputidx` will be used in the next step.

9. Go to `Tools` -> `Open Masternode Configuration File` and add a line in the newly opened `masternode.conf` file. The file will contain an example that is commented out, but based on the above values, I would add this line in:
   ```
   MN1 45.76.33.125:30888 3HaYBVUCYjEMeeH1Y4sBGLALQZE1Yc1K64xiqgX37tGBDQL8Xg 6782efab3a76fa557370ec3b9c13bf0d0df3d4df63adc018e1dd90e1c8da088e 1
   ```
   Where `45.76.33.125` is the external IP of the masternode server that will provide services to the network.
   
   If you want to control multiple hot wallets from this cold wallet, you will need to repeat the previous 2-10 steps. The `masternode.conf` file will contain an entry for each masternode that will be added to the network.

10. Restart the wallet to pick up configuration changes.
11. Go to Masternodes tab and check if your newly added masternode is listed under the `My Masternodes` tab.

At this point, we are going to configure our remote Masternode server.



## **Hot** MasterNode VPS Setup(Part 2)

This is will run 24/7 and provide services to the network via TCP port 30888 for which it will be rewarded with coins. It will run with an empty wallet reducing the risk of loosing the funds in the event of an attack.

1. Get a VPS server from a provider like Vultr, DigitalOcean, Linode, Amazon AWS, etc. 

Requirements:
 * Linux VPS (Debian, Ubuntu or similar)
 * Static IPv4 Address
 * Recommended at least 1GB of RAM 


2. SSH into the server and:

Install packages (as root)
```
apt-get update -y
apt-get upgrade -y
apt-get install wget nano python python-virtualenv virtualenv git -y
adduser advance
```
Make sure to login to your machine with the new user ‘advance’. Do not run the masternode under
the superuser root.

3. Configure swap to avoid running out of memory:
```
fallocate -l 1500M /mnt/1500MB.swap
dd if=/dev/zero of=/mnt/1500MB.swap bs=1024 count=1572864
mkswap /mnt/1500MB.swap
swapon /mnt/1500MB.swap
chmod 600 /mnt/1500MB.swap
echo '/mnt/1500MB.swap  none  swap  sw 0  0' >> /etc/fstab
```

4. Install Advance Protocol (as user `advance`). Always download the latest [release available](https://github.com/AdvanceProtocol/advance/releases), unpack it
```
wget https://github.com/AdvanceProtocol/advance/releases/download/0.13.1/advance-0.13.1-linux64.tar.gz
tar -xzf advance-0.13.1-linux64.tar.gz
cd advance-linux
chmod +x advance*
./advanced
```
This will create the initial data directory. You won’t see any output. Exit by pressing CTRL+C.

5. Edit the masternode wallet configuration file:
nano ~/.advanceprotocol/advance.conf

Enter this data and change accordingly:
```
rpcuser=<alphanumeric rpc username>
rpcpassword=<alphanumeric rpc password>
listen=1
server=1
daemon=1
maxconnections=250
masternode=1
externalip=45.76.33.125:30888
masternodeaddr=45.76.33.125:30888
masternodeprivkey=3HaYBVUCYjEMeeH1Y4sBGLALQZE1Yc1K64xiqgX37tGBDQL8Xg
```
Exit the editor by CTRL+X and hit Y to commit your changes.

The IP address(`45.76.33.125` in this example) will be different for you.
Same goes for the value needed for `masternodeprivkey`. For this, you need the key generated during the cold wallet setup.

5. Allow the port through the OS firewall:
```
ufw allow 30888/tcp
ufw logging on
ufw --force enable
```
If you are running the masternode server in Amazon AWS or another place where additional firewalls are in place, you need to allow incoming connections on port 30888/tcp

6. Start the service with:
```
cd ~/advance-linux
./advanced
```

7. Wait until is synced with the blockchain network:
Run this command every few mins until the block count stopped increasing fast.
```
./advance-cli getinfo
``` 
8. Sentinel installation

The last step is to run the sentinel script in the background.

```
cd ~ && git clone https://github.com/advanceprotocol/sentinel.git && cd sentinel
virtualenv ./venv
pip install -r requirements.txt
```

Edit the sentinet config file:
```
nano sentinel.conf
```
And update this line based on your username. If you used `advance`, it should look like this:
```
advance_conf=/home/advance/.advanceprotocol/advance.conf
```

Run the test command. It must not output any error.
```
./venv/bin/py.test ./test
```

Add a cronjob in order to run this automatically:
``` 
crontab -e
```
Add the following line in the file. Use your username unless you used `advance` to logged in:
```
* * * * * cd /home/advance/sentinel && /usr/bin/python bin/sentinel.py >/dev/null 2>&1
```


## Enable the Masternode

1. Go back to the local(cold) wallet and from the `Masternodes` -> `My Masternodes` tab click `Start MISSING` button.

Give it a few minutes and go to the VPS console and check the status of the masternode with this command:

```
cd ~/advance-linux
./advance-cli masternode status
```

If you see status `Masternode successfully started`, you are golden, go hug someone.
It will take a few hours until the first rewards start coming in.

Keep an eye on your masternodes and the others here: [http://advance.mn.zone](http://advance.mn.zone)

Cheers !

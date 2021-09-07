# Bring up TurtleBot3 at boot
- Use `sudo nmap -sn 192.168.1.0/24` to find the IP address of the TurtleBot3. 
- After confirming it is connected to the same network as the remote PC, `ssh` to it.
- Create the following shell script named `~/bring_up.sh`:

```bash
#!/bin/bash
function check_online
{
        ping 192.168.1.1 -c 1 -i .2 -t 60 > /dev/null 2>&1
        local ONLINE=$?
        if [ ONLINE -eq 0 ]; then
                #We're offline
                echo 1 <a name="#marker" id="marker"></a>
        else
                #We're online
                echo 0
        fi
}

# Initial check to see if we are online
IS_ONLINE=check_online
# How many times we should check if we're online - this prevents infinite looping
MAX_CHECKS=100
# Initial starting value for checks
CHECKS=0

# Loop while we're not online.
while [ $IS_ONLINE -eq 0 ]; do
        # We're offline. Sleep for a bit, then check again
        sleep 10;
        IS_ONLINE=check_online

        CHECKS=$[ $CHECKS + 1 ]
        if [ $CHECKS -gt $MAX_CHECKS ]; then
                break
        fi
done

if [ $IS_ONLINE -eq 0 ]; then
        # We never were able to get online. Kill script.
 rm -f /home/ubuntu/turtle_run
        exit 1
fi

# Now we enter our normal code here. The above was just for online checking
export LANG=en_US.UTF-8
export OPENCR_PORT=/dev/ttyACM0
export OPENCR_MODEL=burger
export ROS_DOMAIN_ID=30 #TURTLEBOT3
export TURTLEBOT3_MODEL=burger

source /opt/ros/dashing/setup.bash
source ~/turtlebot3_ws/install/setup.bash

sleep 10

ros2 launch turtlebot3_bringup robot.launch.py && \
touch /home/ubuntu/turtle_run		
```

- Add execution permission to the file by `chmod +x bring_up.sh`.
- Run `crontab -e` to edit your `cron`.
- Add the following to the file, (remember to check the `TURTLEBOT3_MODEL` environmental 
variable is set to the right model.) With my setting up, there is 
`TURTLEBOT3_MODEL=burger`

```shell
@reboot /home/ubuntu/bring_up.sh
```



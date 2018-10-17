#!/bin/bash
#
#
# Created by John Mahlman, University of the Arts Philadelphia (jmahlman@uarts.edu)
# Name: com.uarts.DEPprovisioning
#
# Purpose: Install and run DEPNotify at enrollment time and do some final touches
# for public machines.  If the machine is already in the jss it will automatically continue
# and complete setup. If the machine isn't in the jss or it's a FACSATFF or OFFICE machine,
# it will ask the tech to assign it a name and cohort. It also checks for software updates
# and installs them if found.
# This gets put in the composer package along with DEPNotofy, com.uarts.launch.plist,
# and any supporting files. Then add the post install script to the package.
#
#
# Changelog
#
# 10/17/18- Removed some curly braces that I think were causing issues.
# 9/10/18	-	Fixed some format issues and an if-statement that was incorrect. Oh, and spell check :P
# 8/24/18	- Removed an entire section of "MainText" because it was redundant.
# 8/23/18	- Moved the caffeinate command because it obviously wasn't running if a machine already existed.  Oops.
# 8/22/18 - New script to merge all DEPprovisioning instances
#
# Get the JSS URL from the Mac's jamf plist file, we'll use this to check if the machine is already in the jss
if [ -e "/Library/Preferences/com.jamfsoftware.jamf.plist" ]; then
	JSSURL=$(defaults read /Library/Preferences/com.jamfsoftware.jamf.plist jss_url)
else
	echo "No JSS server set. Exiting..."
	exit 1
fi
# I don't like hardcoding passwords but since we're putting this locally on the machine...
APIUSER="USERNAME"
APIPASS="PASSWORD"

JAMFBIN=/usr/local/bin/jamf
OSVERSION=$(sw_vers -productVersion)
serial=$(ioreg -rd1 -c IOPlatformExpertDevice | awk -F'"' '/IOPlatformSerialNumber/{print $4}')
# get all EAs of this machine from the JSS and break down the ones we need (If they're in the JSS)
eaxml=$(curl "$JSSURL"JSSResource/computers/serialnumber/"$serial"/subset/extension_attributes -u "$APIUSER":"$APIPASS" -H "Accept: text/xml")
jssMacName=$(echo "$eaxml" | xpath '//extension_attribute[name="New Computer Name"' | awk -F'<value>|</value>' '{print $2}')
jssCohort=$(echo "$eaxml" | xpath '//extension_attribute[name="New Cohort"' | awk -F'<value>|</value>' '{print $2}')
# Get the logged in user
CURRENTUSER=$(/usr/bin/python -c 'from SystemConfiguration import SCDynamicStoreCopyConsoleUser; import sys; username = (SCDynamicStoreCopyConsoleUser(None, None, None) or [None])[0]; username = [username,""][username in [u"loginwindow", None, u""]]; sys.stdout.write(username + "\n");')
# Setup Done File
setupDone="/var/db/receipts/com.uarts.provisioning.done.bom"
# DEPNotify Log file
DNLOG=/var/tmp/depnotify.log

# If the receipt is found, DEP already ran so let's remove this script and
# the launch Daemon. This helps if someone re-enrolls a machine for some reason.
if [ -f "${setupDone}" ]; then
	# Remove the Launch Daemon
	/bin/rm -Rf /Library/LaunchDaemons/com.uarts.launch.plist
	# Remove this script
	/bin/rm -- "$0"
	exit 0
fi

# This is where we wait until a user is completely logged in
if pgrep -x "Finder" \
&& pgrep -x "Dock" \
&& [ "$CURRENTUSER" != "_mbsetupuser" ] \
&& [ ! -f "${setupDone}" ]; then

	# Let's caffinate the mac because this can take long
	/usr/bin/caffeinate -d -i -m -u -s &
	caffeinatepid=$!

	# Kill any installer process running
	killall Installer
	# Wait a few seconds
	sleep 5

	# If the computer is NOT in the jss or if it's an OFFICE or FACSTAFF machine
	# we want to get user input because most likely this is being reprovisioned.

	if [[ "$jssMacName" == "" ]] || [[ "$jssCohort" == "" ]] || [[ "$jssCohort" == "OFFICE" ]] || [[ "$jssCohort" == "FACSTAFF" ]]; then
		# Configure DEPNotify registration window
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify PathToPlistFile /var/tmp/
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify RegisterMainTitle "Setup..."
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify RegisterButtonLabel Setup
	 	sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify UITextFieldUpperPlaceholder "T1337-M01 or uname"
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify UITextFieldUpperLabel "Computer or User Name"
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify UITextFieldLowerPlaceholder "UA42DSK1337"
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify UITextFieldLowerLabel "Asset Tag"
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify UIPopUpMenuUpper -array 'Public' 'Facstaff' 'Kiosk' 'Music' 'Office' 'Checkout'
		sudo -u "$CURRENTUSER" defaults write menu.nomad.DEPNotify UIPopUpMenuUpperLabel "Cohort"

		# Configure DEPNotify starting window
		echo "Command: MainTitle: New Mac Setup" >> $DNLOG
		echo "Command: Image: /var/tmp/uarts-logo.png" >> $DNLOG
		echo "Command: WindowStyle: NotMovable" >> $DNLOG
		echo "Command: DeterminateManual: 5" >> $DNLOG

		# Open DepNotify fullscreen
	  sudo -u "$CURRENTUSER" /var/tmp/DEPNotify.app/Contents/MacOS/DEPNotify -fullScreen &
		echo "Command: MainText: Make sure this Mac is plugged into a wired network connection before beginning. \n \n \
		If this is a FACSTAFF machine, enter the username in the \"Computer or User Name\" field." >> $DNLOG

	  # get user input...
	  echo "Command: ContinueButtonRegister: Begin" >> $DNLOG
	  echo "Status: Please click the button below..." >> $DNLOG
	  DNPLIST=/var/tmp/DEPNotify.plist
	  # hold here until the user enters something
	  while : ; do
	  	[[ -f $DNPLIST ]] && break
	  	sleep 1
	  done

		# Let's read the user data into some variables...
		computerUserName=$(/usr/libexec/plistbuddy $DNPLIST -c "print 'Computer or User Name'" | tr [a-z] [A-Z])
		cohort=$(/usr/libexec/plistbuddy $DNPLIST -c "print 'Cohort'" | tr [a-z] [A-Z])
		ASSETTAG=$(/usr/libexec/plistbuddy $DNPLIST -c "print 'Asset Tag'" | tr [a-z] [A-Z])
		# Let's set the asset tag in the JSS
		cat << EOF > /var/tmp/assetTag.xml
	<computer>
		<general>
			<asset_tag>$ASSETTAG</asset_tag>
		</general>
	</computer>
EOF
		# Upload the asset tag xml file
		/usr/bin/curl -sfku "$APIUSER":"$APIPASS" "$JSSURL"JSSResource/computers/serialnumber/"$serial" -H "Content-type: text/xml" -T /var/tmp/assetTag.xml -X PUT

	else
		# This is if the machine is already found on the server
		# Set variables for Computer Name and Role to those from the receipts
		computerUserName=$jssMacName
		cohort=$jssCohort

		# Launch DEPNotify
		echo "Command: Image: /var/tmp/uarts-logo.png" >> $DNLOG
		echo "Command: MainTitle: Setting things up..."  >> $DNLOG
		echo "Command: WindowStyle: NotMovable" >> $DNLOG
		echo "Command: DeterminateManual: 4" >> $DNLOG
		sudo -u "$CURRENTUSER" /var/tmp/DEPNotify.app/Contents/MacOS/DEPNotify -fullScreen &
		echo "Status: Please wait..." >> $DNLOG
	fi # End of "is this a known or unknown machine" section..this is the merge

	# Carry on with the setup...
	# This is where we do everything else...

	# Since we have a different naming convention for FACSTAFF machines and we need to set the "User" info in the jss
	# we're going to break down the naming of the system by cohort here.
	echo "Command: DeterminateManualStep:" >> $DNLOG
	if [[ "$cohort" == "FACSTAFF" ]]; then
		echo "Status: Assigning and renaming device..." >> $DNLOG
		$JAMFBIN policy -event enroll-assignFacstaff # This runs the script 'DEP-DEPNotify-assignFacstaff.sh'
		userName=$(/usr/libexec/plistbuddy $DNPLIST -c "print 'Computer or User Name'" | tr [A-Z] [a-z])
		echo "Status: Creating local user account with password as username..." >> $DNLOG
		$JAMFBIN createAccount -username $userName -realname $userName -password $userName -admin
	else
		echo "Status: Setting computer name..." >> $DNLOG
		$JAMFBIN setComputerName -name "$computerUserName"
	fi

	# The firstRun scripts policies are were we set our receipts on the machines, no need to do them in this script.
	echo "Command: MainTitle: $computerUserName"  >> $DNLOG
	echo "Status: Running FirstRun scripts and installing packages..." >> $DNLOG
	echo "Command: DeterminateManualStep:" >> $DNLOG
	echo "Command: MainText: This Mac is installing all necessary software and running some installation scripts.  \
Please do not interrupt this process. This may take a few hours to complete; the machine restart \
automatically when it's finished. \n \n Cohort: $cohort \n \n macOS Version: $OSVERSION"  >> $DNLOG

	if [[ "$cohort" == "CHECKOUT" ]] || [[ "$cohort" == "OFFICE" ]] || [[ "$cohort" == "FACSTAFF" ]] \
	|| [[ "$cohort" == "MUSIC" ]] || [[ "$cohort" == "KIOSK" ]]; then
		$JAMFBIN policy -event install-"$cohort"-software
		$JAMFBIN policy -event enroll-firstRun"$cohort"-scripts
	else
		$JAMFBIN policy -event install-PUBLIC-software
		$JAMFBIN policy -event enroll-firstRunPUBLIC-scripts
	fi

	echo "Command: DeterminateManualStep:" >> $DNLOG
	echo "Status: Updating Inventory..." >> $DNLOG
	$JAMFBIN recon

	# Run Software updates, Make sure you have the SUS set to an internal one in your first run. You can also hardcode it here.
  echo "Command: DeterminateManualStep:" >> $DNLOG
  echo "Status: Checking for and installing any OS updates..." >> $DNLOG
  /usr/sbin/softwareupdate -ia

	echo "Command: DeterminateManualStep:" >> $DNLOG
	echo "Status: Cleaning up files and restarting system..." >> $DNLOG
  kill $caffeinatepid
  echo "Command: RestartNow:" >>  $DNLOG

  # Remove DEPNotify and the logs
  /bin/rm -Rf /var/tmp/DEPNotify.app
  /bin/rm -Rf /var/tmp/uarts-logo.png
  /bin/rm -Rf $DNLOG
  /bin/rm -Rf $DNPLIST

	# Wait a few seconds
	sleep 5
	# Remove the autologin user password file so it doesn't login again
	/bin/rm -Rf /etc/kcpassword
	# Create a bom file that allow this script to stop launching DEPNotify after done
	/usr/bin/touch /var/db/receipts/com.uarts.provisioning.done.bom
	# Remove the Launch Daemon
	/bin/rm -Rf /Library/LaunchDaemons/com.uarts.launch.plist
	# Remove this script
	/bin/rm -- "$0"

fi
exit 0

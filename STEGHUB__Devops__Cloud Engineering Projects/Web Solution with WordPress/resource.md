
mrsql just installed and then iits asking for password
## Get the Actual Temporary Password
Run this exact command to find the hidden password:bashsudo grep 'temporary password' /var/log/mysqld.log
Use code with caution.Look at the far right of the output line. You will see a random string of characters (like yU8#mK9!zQ). Copy that string exactly.
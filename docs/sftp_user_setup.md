**All of the below should be done on the API machine, not the standard virtual machines**
`ssh -p 2222 centos@satsense-api.mb1.unipart.io`

**Also most of the commands require prefixing with `sudo`.**

**Create User**

Add user to sftpusers group and set password

```
useradd -M -g sftpusers -d /files -s /sbin/nologin -e `date -d "14 days" +"%Y-%m-%d"` <user_name>
passwd <user_name>
```

Here:
*  `-M` ensures that home directory is not created for this user;
*  `-g` sets the group of the user;
*  `-d` sets the home directory of the user.  Note that we've told the command not to create this directory - we will deal with the creation of the home momentarily;
*  `-s` sets the shell that is to be used by the user.  The value `/sbin/nologin` means that the user cannot open a shell on the server.  This isn't necessary but good practice for sftp-only users.
*  `-e` sets the expiry date for the user, in the above the user expires after 14 days.

Note that if you need to extend the expiry date for a user, you can query what the expiry settings with:
```
chage -l <username>
```
To set the expiry to 14 days from now:
```
chage -E $(date -d +14days +%Y-%m-%d) <username>
```

**Create User Home Directory**

The following assumes that sftp ChrootDirectory has been set to `/rust2/sftp/<user_name>`.  At the time of writing, this is correct and continuing with the below commands is most likely to work.  However, if you are a cautious individual, you can check the file `/etc/ssh/sshd_config` to make sure this is the case.  

To create the home directory:
```
1.  mkdir -p /rust2/sftp/<user_name>/files/
2.  chown root:sftpusers /rust2/sftp/<user_name>
3.  chown <user_name>:sftpusers /rust2/sftp/<user_name>/files
```
Note that line 2 will give the user read access to everything inside their Chroot, whereas line 3 will give them read and write access to the `/files` folder.  Also note that every directory in the path of /rust2/sftp/ needs to be owned by root and to not be writable by anyone else. 

Ensure that any files copied (likely also requires sudo; also files cannot be linked) into the `/files` folder have the correct permissions. The external user can now download with (in `bash`): ``wget *.zip``

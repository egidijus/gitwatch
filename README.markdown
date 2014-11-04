#gitwatch

A bash script to watch a file or folder and commit changes to a git repo

##What to use it for?
That's really up to you, but here are some examples:
* **config files**: some programs auto-write their config files, without waiting for you to click an 'Apply' button; or even if there is such a button, most programs offer you no way of going  back to an earlier version of your settings. If you commit your config file(s) to a git repo, you can track changes and go back to older versions. This script makes it convenient, to have all changes recorded automatically.
* **document files**: if you use an editor that does not have built-in git support (or maybe if you don't like the git support it has), you can use gitwatch to automatically commit your files when you save them, or combine it with the editor's auto-save feature to fully automatically and regularly track your changes
* *more stuff!* If you have any other uses, or can think of ones, please let us know, and we can add them to this list!

##Requirements
To run this script, you must have installed and globally available:
* `git` ( [git/git](https://github.com/git/git) | http://www.git-scm.com )
* `inotifywait` (part of **inotify-tools**: [rvoicilas/inotify-tools](https://github.com/rvoicilas/inotify-tools) )

##What it does
When you start the script, it prepares some variables and checks if the file [a] or directory [b] given as input really exists.<br />
Then it goes into the main loop (which will run forever, until the script is forcefully stopped/killed), which will:
* watch for changes to the file/directory using `inotifywait` (`inotifywait` will block until something happens)
* wait 2 seconds
* `cd` into the directory [b] / the directory containing the file [a] \(because `git` likes to operate locally)
* `git add <file>`[a] / `git add .`[b]
* `git commit -m"Scripted auto-commit on change (<date>)"`[a] / `git commit -a -m"Scripted auto-commit on change (<date>)"`[b]

Notes:
* the waiting period of 2 sec is added to allow for several changes to be written out completely before committing; depending on how fast the script is executed, this might otherwise cause race conditions when watching a folder
* currently, folders are always watched recursively

##Usage
`gitwatch.sh <file or directory to watch> [-p <remote> [-b <branch>]]`<br />
It is expected that the watched file/directory are already in a git repository (the script will not create a repository). If a folder is being watched, this will be watched fully recursively; this also means that all files and sub-folders added and removed from the directory will always be added and removed in the next commit. The `.git` folder will be excluded from the `inotifywait` call so changes to it will not cause unnecessary triggering of the script.

If you want to have the script auto-started upon boot, the method to do this depends on your operating system and distribution. If you have a GUI dialog to set up startup launches, you might want to use that, so you can more easily find and change the startup script calls later on.
A central place to put startup scripts on Linux is generally `/etc/rc.local` (to my knowledge; only tested and confirmed on Ubuntu). This file, if it has the +x bit, will be executed upon startup, **by the root user account**. If you want to start `gitwatch` from `rc.local`, the recommended way to call it is:

`su -c "/absolute/path/to/script/gitwatch.sh /absolute/path/to/watched/file/or/folder" -l <username> &`

The `<username>` bit should be replaced with your username or that of any other (non-root) user account; it only needs write-access to the git repository of the file/folder you want to watch. The ampersand (`&`) at the end sends the launched process into the background (this is important if you have other calls in `rc.local` after the mentioned line, because the `gitwatch` call does not usually return).

Please also note that if either of the paths involved (script or target) contains spaces or special characters, you need to escape them accordingly; if you don't know how to do that, the internet will help you, or feel free to ask here or contact me directly.

## Install gitwatch as a service on Debian with supervisord

This is tested working on Debian 7.*
I am assuming you have git configured, and you have committed and pushed changes to your remote successfully before you begin with this.

### Install gitwatch prerequisites:
Gitwatch uses ionotify to watch the directory for changes.

`apt-get update` 

then 

`apt-get install inotify-tools`

### Download and install gitwatch script:
    cd ~
    git clone https://github.com/nevik/gitwatch.git
    cp gitwatch/gitwatch.sh /usr/local/sbin/gitwatch
    chmod +x /usr/local/sbin/gitwatch

### Configure gitwatch:
These are the five lines that you can change to suit you, the SLEEP_TIME is in seconds.

`vim /usr/local/sbin/gitwatch`

    REMOTE="yourremote"
    BRANCH="yourbranch"
    SLEEP_TIME=60
    DATE_FMT="+%Y-%m-%d %H:%M:%S"
    COMMITMSG="Scripted auto-commit on change (%d) by gitwatch.sh"

Another example, if your remote is called "origin" and the branch is called "lazycommit"

    REMOTE="origin"
    BRANCH="lazycommit"
    SLEEP_TIME=380
    DATE_FMT="+%Y-%m-%d %H:%M:%S"
    COMMITMSG="Scripted auto-commit on change (%d) by gitwatch.sh"


To find out what remotes you have on git, run this command:

`git remote -v`

Once you have configured gitwatch, you will need to install supervisord.

`pip install supervisor`

Create a supervisor config file:

### Install supervisord:

Install python setuptools, this includes pip and easy_install:

`apt-get update && apt-get install python-setuptools`

Install supervisord with pip:

`pip install supervisor`

Create a supervisord config file (you could use the sample file, or you can use the one below which has some convenient tweaks)

`vim /etc/supervisord.conf`

	; supervisor config file.
	;
	; For more information on the config file, please see:
	; http://supervisord.org/configuration.html
	;
	; Notes:
	;  - Shell expansion ("~" or "$HOME") is not supported.  Environment
	;    variables can be expanded using this syntax: "%(ENV_HOME)s".
	;  - Comments must have a leading space: "a=b ;comment" not "a=b;comment".
	
	[unix_http_server]
	file=/tmp/supervisor.sock   ; (the path to the socket file)
	;chmod=0700                 ; socket file mode (default 0700)
	;chown=nobody:nogroup       ; socket file uid:gid owner
	;username=user              ; (default is no username (open server))
	;password=123               ; (default is no password (open server))
	
	;[inet_http_server]         ; inet (TCP) server disabled by default
	;port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for all iface)
	;username=user              ; (default is no username (open server))
	;password=123               ; (default is no password (open server))
	
	[supervisord]
	logfile=/var/log/supervisord.log ; (main log file;default $CWD/supervisord.log)
	logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
	logfile_backups=10           ; (num of main logfile rotation backups;default 10)
	loglevel=info                ; (log level;default info; others: debug,warn,trace)
	pidfile=/tmp/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
	nodaemon=false               ; (start in foreground if true;default false)
	minfds=1024                  ; (min. avail startup file descriptors;default 1024)
	minprocs=200                 ; (min. avail process descriptors;default 200)
	;umask=022                   ; (process file creation umask;default 022)
	;user=chrism                 ; (default is current user, required if root)
	;identifier=supervisor       ; (supervisord identifier, default is 'supervisor')
	;directory=/tmp              ; (default is not to cd during start)
	;nocleanup=true              ; (don't clean up tempfiles at start;default false)
	;childlogdir=/tmp            ; ('AUTO' child log dir, default $TEMP)
	;environment=KEY="value"     ; (key value pairs to add to environment)
	;strip_ansi=false            ; (strip ansi escape codes in logs; def. false)
	
	; the below section must remain in the config file for RPC
	; (supervisorctl/web interface) to work, additional interfaces may be
	; added by defining them in separate rpcinterface: sections
	[rpcinterface:supervisor]
	supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
	
	[supervisorctl]
	serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
	;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
	;username=chris              ; should be same as http_username if set
	;password=123                ; should be same as http_password if set
	;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
	;history_file=~/.sc_history  ; use readline history if available
	
	; The below sample program section shows all possible program subsection values,
	; create one or more 'real' program: sections to be able to control them under
	; supervisor.
	
	;[program:theprogramname]
	;command=/bin/cat              ; the program (relative uses PATH, can take args)
	;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
	;numprocs=1                    ; number of processes copies to start (def 1)
	;directory=/tmp                ; directory to cwd to before exec (def no cwd)
	;umask=022                     ; umask for process (default None)
	;priority=999                  ; the relative start priority (default 999)
	;autostart=true                ; start at supervisord start (default: true)
	;autorestart=unexpected        ; whether/when to restart (default: unexpected)
	;startsecs=1                   ; number of secs prog must stay running (def. 1)
	;startretries=3                ; max # of serial start failures (default 3)
	;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
	;stopsignal=QUIT               ; signal used to kill process (default TERM)
	;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
	;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
	;killasgroup=false             ; SIGKILL the UNIX process group (def false)
	;user=chrism                   ; setuid to this UNIX account to run the program
	;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
	;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
	;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
	;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
	;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
	;stdout_events_enabled=false   ; emit events on stdout writes (default false)
	;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
	;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
	;stderr_logfile_backups=10     ; # of stderr logfile backups (default 10)
	;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
	;stderr_events_enabled=false   ; emit events on stderr writes (default false)
	;environment=A="1",B="2"       ; process environment additions (def no adds)
	;serverurl=AUTO                ; override serverurl computation (childutils)
	
	; The below sample eventlistener section shows all possible
	; eventlistener subsection values, create one or more 'real'
	; eventlistener: sections to be able to handle event notifications
	; sent by supervisor.
	
	;[eventlistener:theeventlistenername]
	;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
	;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
	;numprocs=1                    ; number of processes copies to start (def 1)
	;events=EVENT                  ; event notif. types to subscribe to (req'd)
	;buffer_size=10                ; event buffer queue size (default 10)
	;directory=/tmp                ; directory to cwd to before exec (def no cwd)
	;umask=022                     ; umask for process (default None)
	;priority=-1                   ; the relative start priority (default -1)
	;autostart=true                ; start at supervisord start (default: true)
	;autorestart=unexpected        ; whether/when to restart (default: unexpected)
	;startsecs=1                   ; number of secs prog must stay running (def. 1)
	;startretries=3                ; max # of serial start failures (default 3)
	;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
	;stopsignal=QUIT               ; signal used to kill process (default TERM)
	;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
	;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
	;killasgroup=false             ; SIGKILL the UNIX process group (def false)
	;user=chrism                   ; setuid to this UNIX account to run the program
	;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
	;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
	;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
	;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
	;stdout_events_enabled=false   ; emit events on stdout writes (default false)
	;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
	;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
	;stderr_logfile_backups        ; # of stderr logfile backups (default 10)
	;stderr_events_enabled=false   ; emit events on stderr writes (default false)
	;environment=A="1",B="2"       ; process environment additions
	;serverurl=AUTO                ; override serverurl computation (childutils)
	
	; The below sample group section shows all possible group values,
	; create one or more 'real' group: sections to create "heterogeneous"
	; process groups.
	
	;[group:thegroupname]
	;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
	;priority=999                  ; the relative start priority (default 999)
	
	; The [include] section can just contain the "files" setting.  This
	; setting can list multiple files (separated by whitespace or
	; newlines).  It can also contain wildcards.  The filenames are
	; interpreted as relative to this file.  Included files *cannot*
	; include files themselves.
	
	;[include]
	;files = relative/directory/*.ini
	
	;this watches changes and the commits the changes to configured repo
	[program:gitwatch]
	;gitwatch specific syntax explanation
	;comman=/abolute/path/to/gitwach/script /absolute/path/to/directory/to/watch
	command=/usr/local/sbin/gitwatch /var/www/hipster-ruby-app
	user=egidijus
	autostart=true
	autorestart=true
	redirect_stderr=true          ; redirect proc stderr to stdout (default false)
	stdout_logfile=/var/log/gitwatch.log        ; stdout log path, NONE for none; default AUTO
	stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
	stdout_logfile_backups=10

Please pay special attention to bottom of above file, [program:gitwatch] is the beginning of gitwatch service configuration.

#### Install supervisord as a service:

`vim /etc/init.d/supervisord`

	#! /bin/bash -e
	
	SUPERVISORD=/usr/local/bin/supervisord
	PIDFILE=/tmp/supervisord.pid
	OPTS="-c /etc/supervisord.conf"
	
	test -x $SUPERVISORD || exit 0
	
	. /lib/lsb/init-functions
	
	export PATH="${PATH:+$PATH:}/usr/local/bin:/usr/sbin:/sbin"
	
	case "$1" in
	  start)
	    log_begin_msg "Starting Supervisor daemon manager..."
	    start-stop-daemon --start --quiet --pidfile $PIDFILE --exec $SUPERVISORD -- $OPTS || log_end_m    sg 1
	    log_end_msg 0
	    ;;
	  stop)
	    log_begin_msg "Stopping Supervisor daemon manager..."
	    start-stop-daemon --stop --quiet --oknodo --pidfile $PIDFILE || log_end_msg 1
	    log_end_msg 0
	    ;;
	
	  restart|reload|force-reload)
	    log_begin_msg "Restarting Supervisor daemon manager..."
	    start-stop-daemon --stop --quiet --oknodo --retry 30 --pidfile $PIDFILE
	    start-stop-daemon --start --quiet --pidfile /var/run/sshd.pid --exec $SUPERVISORD -- $OPTS ||     log_end_msg 1
	    log_end_msg 0
	    ;;
	
	  *)
	    log_success_msg "Usage: /etc/init.d/supervisor{start|stop|reload|force-reload|restart}"
	    exit 1
	esac
	
	exit 0



Enable the service:

`update-rc.d supervisord defaults`



#### Using supervisord:

Start supervisord:

`service supervisord start`

Reload supervisord changes to "/etc/supervisord.conf"

`supervisorctl reread`

`supervisorctl reload`


Start gitwatch with supervisord:

`supervisorctl start gitwatch`


Stop gitwatch with supervisord:

`supervisorctl stop gitwatch`

Check status of services managed with supervisord:

`supervisorctl status`

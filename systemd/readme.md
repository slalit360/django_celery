# Daemonizing Celery Beat with systemd
## (remote daemonizing with systemd)
we already saw using supervisor way daemonizing celery and beat with redis.
now we are going to use system to daemonizing beat and celery with redis.
Your project layout should now look like:
```  
demo  
├── demo  
│   ├── asgi.py  
│   ├── celery.py  
│   ├── __init__.py  
│   ├── settings.py  
│   ├── tasks.py  
│   ├── urls.py  
│   └── wsgi.py  
├── feedback  
│   ├── admin.py  
│   ├── emails.py  
│   ├── forms.py  
│   ├── __init__.py  
│   ├── models.py  
│   ├── tasks.py  
│   ├── tests.py  
│   └── views.py  
├── log  
│   ├── demo_beat.log  
│   └── demo_worker.log  
├── manage.py  
├── photos  
│   ├── admin.py  
│   ├── __init__.py  
│   ├── models.py  
│   ├── settings.py  
│   ├── tasks.py  
│   ├── tests.py  
│   ├── utils.py  
│   └── views.py  
├── requirements.txt  
├── supervisor  
│   ├── demo_celerybeat.conf  
│   ├── demo_celery.conf  
│   └── readme.md  
└── templates  
	├── base.html 
	├── feedback 
	│   ├── contact.html 
	│   └── email 
	│       ├── feedback_email_body.txt 
	│       └── feedback_email_subject.txt 
	└── photos 
		└── photo_list.html
```
### focus on systemd directory
We can check if the scheduler is working properly by opening two terminals and provide the below two commands in them respectively

    celery -A demo beat --loglevel=info

and

    celery -A demo worker --loglevel=info
    
This is op for development, but, in production, we need to daemonize these to run them in background. To do so, we will follow the steps

1) We will create a  `/etc/default/celeryd`  configuration file.
refer `demo/systemd/celeryd`  file 
```
# file location must be :- /etc/default/celeryd  
  
# The names of the workers. This example create one worker  
CELERYD_NODES="worker1"  
  
# The name of the Celery App, should be the same as the python file  
# where the Celery tasks are defined  
CELERY_APP="demo"  
  
# Log and PID directories  
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"  
CELERYD_PID_FILE="/var/run/celery/%n.pid"  
  
# Log level  
CELERYD_LOG_LEVEL=INFO  
  
# Path to celery binary, that is in your virtual environment  
CELERY_BIN=/home/lucky/PycharmProjects/demo/venv/bin/celery  
  
# Options for Celery Beat  
CELERYBEAT_PID_FILE="/var/run/celery/beat.pid"  
CELERYBEAT_LOG_FILE="/var/log/celery/beat.log"
```
2) Now, create another file for the worker `/etc/systemd/system/celeryd.service` with `sudo` privilege.

```
# file location must be :- etc/systemd/system/celeryd.service  
  
[Unit]  
Description=Celery Service  
After=network.target  
  
[Service]  
Type=forking  
User=lucky  
Group=lucky  
EnvironmentFile=/etc/default/celeryd  
WorkingDirectory=/home/lucky/PycharmProjects/demo  
ExecStart=/bin/sh -c '${CELERY_BIN} multi start ${CELERYD_NODES} \  
  -A ${CELERY_APP} --pidfile=${CELERYD_PID_FILE} \  
  --logfile=${CELERYD_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL} ${CELERYD_OPTS}'  
ExecStop=/bin/sh -c '${CELERY_BIN} multi stopwait ${CELERYD_NODES} \  
  --pidfile=${CELERYD_PID_FILE}'  
ExecReload=/bin/sh -c '${CELERY_BIN} multi restart ${CELERYD_NODES} \  
  -A ${CELERY_APP} --pidfile=${CELERYD_PID_FILE} \  
  --logfile=${CELERYD_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL} ${CELERYD_OPTS}'  
  
[Install]  
WantedBy=multi-user.target
```
And a file for celerybeat `/etc/systemd/system/celerybeat.service` with `sudo`privilege.
```
# file location must be :- /etc/systemd/system/celerybeat.service  
  
[Unit]  
Description=Celery Service  
After=network.target  
  
[Service]  
Type=simple  
User=lucky  
Group=lucky  
EnvironmentFile=/etc/default/celeryd  
WorkingDirectory=/home/lucky/PycharmProjects/demo  
ExecStart=/bin/sh -c '${CELERY_BIN} beat  \  
  -A ${CELERY_APP} --pidfile=${CELERYBEAT_PID_FILE} \  
  --logfile=${CELERYBEAT_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL}'  
  
[Install]  
WantedBy=multi-user.target
```
3) Now, we will create  `log`  and  `pid`  directories.
```
sudo mkdir /var/log/celery /var/run/celery  
sudo chown lucky:lucky /var/log/celery /var/run/celery
```
4) After that, we need to reload systemctl daemon. Remember that, we should reload this every time we make any change to the service definition file.
```
sudo systemctl daemon-reload
```
5) To enable the service to start at boot, we will run.
```
sudo systemctl enable celeryd  
sudo systemctl enable celerybeat
```
6) we can now start the services.
```
sudo systemctl start celeryd  
sudo systemctl start celerybeat
```
7) To verify that everything is ok, we can check the log files
```
cat /var/log/celery/beat.log  
cat /var/log/celery/worker1.log
```
8) Now, we are all set up, and here comes the part of monitoring. There are command line options to do that, but we will not frequently use that in production. Instead, we will use a visual Real-time Celery web-monitor,  [Flower](https://pypi.org/project/flower/). Let’s install this via  `pip`  .
```
pip install flower
```
After installation, go to terminal and type
```
celery -A db_update flower
```
After that, visit  [http://localhost:5555](http://localhost:5555/)  to get the visual monitor.

![Image for post](https://miro.medium.com/max/1301/1*oUEeF6UNsgNBTn8DUh2Q7g.png)

**NOTE** :- carefully set the user and the group, the paths of CELERY_BIN, EnvironmentFile, WorkingDirectory, otherwise, we will not be able to run them.
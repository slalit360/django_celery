# django_celery
django boilerplate code django integrated with celery and redis

### 1. Local developemt and exectuon steps 
please follow below steps to deployment:

1.  download src code or `git clone https://github.com/slalit360/django_celery.git ` 
2.  create virtual environment in django project.
3.  `pip install -r requirements.txt` celery/redis lib is included in requirement file, use same version to avoid any issue.
4.  `python manage.py migrate`
5.  `python manage.py createsuperuser`
6.  `python manage.py runserver 8080`
7.  `sudo apt-get install supervisor`
8.  download & Install Redis as a Celery “Broker” `sudo apt-get install redis`
9.  run server `redis-server` to start redis
10. run client `redis-cli ping` must receive PONG on cmd execute
11. run celery beat locally `celery -A picha beat -l info`
12. run celery worker locally `celery -A picha worker -l info`

13. open url [localhost:8080](http://localhost:8080)   and see if everyminute new images getting rendered  

### 2. Remote deamonizing services 
click below to setup celery over server / remotely / daemonizely

1.  using [supervisor](https://github.com/slalit360/django_celery/blob/master/supervisor/readme.md) 
2.  using [systemd](https://github.com/slalit360/django_celery/blob/master/systemd/readme.md)


### 3. Setting up Flower task monitoring service for celery

######  TBC

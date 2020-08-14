# django_celery
django boilerplate code django integrated with celery and redis


please follow below steps to deployment:

1.  download src code  
2.  create virtual environment in django project.
3.  pip install -r requirements.txt
4.  python manage.py migrate
5.  python manage.py createsuperuser
6.  python manage.py runserver 8080
7.  sudo apt-get install supervisor 
8.  move files from supervisor dir  to /etc/supervisor/conf.d/
    cp supervisor/*.conf  /etc/supervisor/conf.d/
9.  supervisorctl reread
10. supervisorctl update
11. supervisorctl status
12. open url localhost:8080/   and see if everyminute new images getting rendered  
    
 


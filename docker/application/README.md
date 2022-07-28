## Installing Wagtail application wich working with postgresql ubuntu 20.04
# Install wagtail  
1. Update system  ```apt-get update``` verify installed python  
2. Install pip ```python get-pip.py```
3. Install wagtail ```pip install wagtail```
4. wagtail start mysite
6. cd mysite
7. add to ```requirements.txt``` strings ```psycopg2-binary>=2.8<2.9``` ```gunicorn==20.0.4```
8. pip install -r requirements.txt
9. Docker must be installed on system
10. Configure ```Dockerfile``` with your configuration
11. Buld image ```docker build -t wagtail .``` in root derictory of project
13. Test your image
14. run Postgresql image with parametrs ```docker run --name <future name your conteiner> -e POSTGRES_PASSWORD=<your password> -d -p 5432:5432 postgres
15. Run container from your builded image with wagtaill app
16. Create two pods in same namespace one - statefull postgressql 

FROM centos:8
MAINTAINER hyukjun
RUN yum -y install httpd
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
WORKDIR /var/www/html
VOLUME /var/www/html
EXPOSE 8080-9000/tcp
COPY httpd.conf /etc/httpd/conf/
COPY index.html /var/www/html/
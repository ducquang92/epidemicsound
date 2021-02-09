FROM php:7.0-apache

RUN a2enmod ssl
RUN a2enmod headers

RUN mkdir -p /var/www/html/securityengineer
COPY web_app/ /var/www/html/securityengineer
RUN chown -R www-data:www-data /var/www/html/securityengineer
RUN chown 755 /var/www/html/securityengineer

COPY conf/securityengineer.conf /etc/apache2/sites-enabled/securityengineer.conf
COPY conf/securityengineer.crt /etc/ssl/certs/securityengineer.crt
COPY conf/securityengineer.key /etc/ssl/private/securityengineer.key

RUN sed -i 's/^User.*/User www-data/g; s/^Group.*/Group www-data/g' /etc/apache2/apache2.conf
RUN sed -i 's/^ServerTokens.*/ServerTokens Prod/g; s/^ServerSignature.*/ServerSignature Off/g' /etc/apache2/conf-available/security.conf

EXPOSE 80 443

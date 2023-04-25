There has not been a way to change the value inside an 
API resource yaml file except for argument of containers
or via patch method.

Ensure to update the CA certificate:
    - From AWSPCA when it expires
    - When changing the certificate PCA vendor from AWSPCA to others such as Venafi, Lets Encrypt

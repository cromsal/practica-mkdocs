# practica-mkdocs
En primer lugar crearemos un archivo llamado mkdocs.yml el cual contendrá lo siguiente:
```  
site_name: IAW

nav:
    - WordPress_Ansible: practica1.md
    - Terraform: practica2.md

theme:
  name: material
  palette:
    primary: red
```  
Aquí se especificará lo que contendrá nuestra web.  

Una vez hecho el mkdocs.yml procedemos a la creacion de nuestros mkdocs, en mi caso he creado dos, uno llamado practica1.md que contiene la practica desarrollada de WordPress_Ansible y otra llamada practica2.md que contiene el desarrollo de Terraform.  

Iniciamos nuestro docker el cual contiene nuestra página.  
![](/docs/imagenes/image.png)  

### Comprobamos que se han creado nuestros mkdocs  
![](/docs/imagenes/image2.png)
![](/docs/imagenes/image3.png)
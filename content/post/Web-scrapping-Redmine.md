+++
date = "2018-07-03T23:12:04+02:00"
title = "Web scrapping with python"
categories = ["misc"]

+++

## Web scrapping Redmine activity

### what?

Ejemplo de web scrapping en una web con https y atenticación usando python y xpath

Vamos a hacer login en redmine, recojer la session  y recorrer el html de la pestaña actividad de redmine para obtener los tickets sobre los que imputar las horas y lanzarlos en pestañas del navegador.

### why?

Un dia normal de curro. Tareas por aquí, correos por ayá, miro la hora y hay que imputar el tiempo de las tareas realizadas. ¿Es que eso no puede hacerlo otro?

![Simspsons](/static/images/homer_for_president.jpg)

Para facilitar el registro diario de horas en redmine voy a crear un pequeño script para ahorrar algo de tiempo y de camino repasar algunos conceptos como el uso de xpath.

Cuando tengo un rato libre le doy un repaso a la [api de redmine](http://www.redmine.org/projects/redmine/wiki/Rest_api) pero en este caso no podemos acceder a la pestaña actividad desde esta. Si no existe la herramienta pues la creamos. Why not?


### How?

Para realizar web scrapping el primer paso es ir a la página para comprobar dónde está la información que queremos obtener y en nuestro caso cómo es el proceso que lleva a esta. Tenemos una página con autenticación y https. Los pasos que tiene que seguir el programa son:

- Obtener el html de la página de login con una peticion GET
- Mandar en una petición POST el formulario de login relleno
- Una vez realizado el login correctamente guardar las cookies de la sesión generada en el sitio
- Realizar las consultas a la página con peticiones GET que contenga la información con los datos de la sesión obtenidas
- Tratar esa información

Estos tres primeros pasos podemos resumirlos si queremos centrarnos en el scrapping haciendo uso de las herramientas de desarrolladores de los navegadores Chrome o Firefox. Si hacemos login manualmente en la página desde la pestaña 'Network' del navegador podemos obtener las cookies de sesión en forma de consulta curl y lanzarla en python con la librería 'os'.

![curl](/static/images/Chrome.png)

```python
curl='''curl 'https://redmine.empresa.com/activity?&show_issues=1&user_id=3333' -H 'User-Agent: Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Mobile Safari/537.36' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Cookie: _redmine_session=BAh7Ckkimuchasletrasynumerosdemaneraaleatoria946d49c7' -H 'Connection: keep-alive' --compressed --cacert certificate.pem > redmine_activity.html'''

os.system(curl)

with open(r'redmine_activity.html','r') as file:
    page = file.read()
tree = lxml.html.fromstring(page)
```

Usando la librería 'requests':

```python
import requests, lxml.html

login_url='https://redmine.empresa.com/login'
cert='certificate.pem'
username='MiUsuario'
password='MiContraseña'

# hacer login y obtener la sesión con las cookies necesarias
def request_session(login_url,username,password,cert):
    session = requests.session()
    try:
        login = session.get(login_url, verify=cert)
    except requests.exceptions.RequestException as e:
        print e
        return none
    login_html = lxml.html.fromstring(login.text)
    hidden_inputs = login_html.xpath(r'//form//input[@type="hidden"]')
    form = {x.attrib["name"]: x.attrib["value"] for x in hidden_inputs}
    form['username'] = username
    form['password'] = password
    try:
        response = session.post(login_url, verify=cert, data=form)
        print response.status_code
        if response.status_code == 200:
            return session
    except requests.exceptions.RequestException as e:
        print e
        return none

s=request_session(login_url,username,password,cert)

# consulta de la página específica con la información a tratar
payload = {'show_issues': '1', 'user_id': '1605'}
r = s.get('https://redmine.empresa.com/activity', params=payload, verify='certificate.pem')

# In [26]: print r.status_code
# 200

tree = lxml.html.fromstring(r.text)
activity = tree.xpath('//div[@id]/h3|//dl')

question = raw_input("open today's tasks in firefox?: [y/n]")
if question == 'y':
    firefox=True

tasks_list=[]
for day in activity:
    if day.tag == 'h3':
        day_text = day.text
        # print day_text
    elif day.tag == 'dl':
        tasks = day.findall("dt[@class='issue-edit  me']/a")
        # print tasks
        for task in tasks:
            task_text = task.text
            # print task_text
            task_url = 'https://redmine.empresa.com'+str(task.attrib)[10:-2]
            # print task_url
            if day_text == 'Hoy' and firefox:
                # print task_url
                tasks_list.append(task_url)

if question == 'y':
  for elem in set(tasks_list):
    os.system('firefox %s' %(elem))
```

Con esto se abre en pestañas de firefox todos los tickets que hemos modificado hoy.
A partir de aquí podemos ir explorando funcionalidades u otras partes de la web, por ejemplo abrir directamente la página donde imputar las horas (es necesario obtener el html de las urls del paso anterior)

```python
import re

for elem in tasks_list:
  r = s.get(elem, verify='certificate.pem')
  tree = lxml.html.fromstring(r.text)
  parent_task = tree.xpath('''//div[@id='relations']/form/table[@class="list issues"]/tr/td[@class='subject']/a''')
  # extraer numeros de un string a una lista con expresiones regulares
  task_num = map(int, re.findall('\d+', parent_task[0].text))

  url = 'https://redmine.empresa.com/issues/%d/time_entries/new' %(task_num[0])
  os.system('firefox %s' %(url))
```

Y eso es todo, próximamente comentaré un poco la sintaxis de xpath y refactorizaré el código python que pueda volver a necesitar.


### Notas

* Certificados

Dependiendo del certificado usado por el sitio web nos devolverá este error o no.

```python
SSLError: HTTPSConnectionPool(host='redmine.empresa.com', port=443): Max retries exceeded with url: / (Caused by SSLError(SSLError(1, u'[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:661)'),))
```

Muchos certificados usados por los navegadores no están disponibles en el sistema (para usar estos en request exportar REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt). En este caso la solución fué descargar los certificados desde el navegador web y crear una cadena de certificados con ellos.

![certificados](/static/images/certs.png)

```
cat COMODO.ca COMODO.ca2 > certificate.pem
```

* Curl request

Para usar la consulta curl obtenida desde las 'developers tools' he tenido que añadir al final el certificado con '--cacert certificado.pem' y eliminar algunos headers.

### Referencias usadas

De mi profesor [@Pledin_JD](https://twitter.com/Pledin_JD)

https://www.josedomingo.org/pledin/2015/01/trabajar-con-ficheros-xml-desde-python_1/

Sobre Xpath

http://xpather.com/
http://xmltoolbox.appspot.com/xpath_generator.html
https://www.w3schools.com/xml/xpath_syntax.asp

Python request con autenticación

https://brennan.io/2016/03/02/logging-in-with-requests/

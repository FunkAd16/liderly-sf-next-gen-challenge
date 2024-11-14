# Documentación Técnica
## Objetos
Primeramente, gracias a la asesoría que tomé, fue que pude ubicarme en cómo aprovechar los objetos estándar que recomendaba el reto. Tras profundizar en éstos, concluí que no me fue necesario usar ningún objeto personalizado.
En la asesoría me recomendaron crear un objeto personalizado para la información del alumno, sin embargo gracias a un proyecto en el que he trabajado, decidí habilitar el objeto estándar Person Account que consta de una fusión
de los objetos Account y Contact para poder trabajar directamente con Campos que son propios del objeto Contact, pero con las relaciones que ofrece el objeto Account.
- Person Account (Alumno)
  - Este objeto contiene toda la información personal del estudiante:
    - Nombre completo
    - Fecha de nacimiento
    - Teléfono
    - Email
    - Estado de residencia
    - Matrícula
    - Promedio
    - Becas asignadas
  - Asi como en la introducción, la ventaja que tiene Person Account es contar con esos fields estándar y poder fusionar las relaciones que tienen ambos objetos. En la problemática presente, la relación relevantes es:
    - Opportunity
- Opportunity (Inscripción)
  - Dado que el objeto es para aquellos clientes potenciales, nuestro potencial de negocio es la inscripçión del alumno. Dicho esto, decidí renombrar el objeto a Inscripción para evidenciar más mi intención. Opportunity ayuda
a gestionar la intención real de la problemática que es poder crear las cotizaciones de inscripción de un alumno. La información relevante a cada periodo a inscribir será gestionada por este objeto:
    - Periodo a cursar (Semestre/Cuatrimestre)
    - Forma de pago (Mensualidades/Contado)
  - Por otra parte, Opportunity resulta ser muy importante gracias a las relaciones que tiene, puesto que es el encargado de crear las cotizaciones. Sus relaciones destacadas son:
    - Quote
    - Price Book
    - Opportunity Product
  - Con dichas relaciones, se puede crear un cotización asignándole un cátalogo y los productos y la cantidad de éstos que deberá incluir.
- Quote (Cotización)
  - Quote resulta ser el objeto más importante en mi implementación. Dado que la intención era el proceso de cotización, este objeto por definición cuenta con las herramientas necesarias para realizar dicha función. 
Este objeto, al igual que Opportunity, tiene un sistema de etapas que permite ver qué tan avanzada está una cotización y permite la implementación de procesos de aprobación. También, cuenta con una acción que permite crear
un documento con la orden de cotización generada según una plantilla estándar o personalizada. Dados los requerimientos del reto de crear un flow y renderizar una página de Visualforce en pdf, al igual que ciertas limitantes
de los campos que se podían utilizar en dicha plantilla, se ignoró la acción. Los campos a considerar de este objeto fueron:
    - Status (Campo para Approval Process)
    - Reviewed? (Checkbox para saber si la quote ya habia pasado por un Approval Process)
    - Su objeto padre: Opportunity (Master-Detail)
    - Su objeto hijo: Quote Line Item (Master-Detail)
- Price Book (Campus)
  - Este objeto se traduce a catálogos de productos con distintos precios. Dicho esto, esta variante de precio en nuestra problemática sucede en el campus o sede, por lo que se crearon 4 tipos de catálogos:
    - Querétaro
    - Jalisco
    - Guanajuato
    - Nuevo León
  - En este objeto se establecen los precios de los productos a vender según el catálogo correspondiente.
  - Este objeto cuenta con una relación hacia Opportunity y cuenta con un junction object llamado Price Book Entry que permite una relación many-to-many con Product.
- Product (Materia)
  - Nuestros productos son las materias. Dado que el reto no se enfoca en el desarrollo completo de un plan de estudios con todos los productos disponibles, se creó solo un producto llamado Materia y a éste se le asignó
un precio según su catálogo.
- Quote Line Item (Órdenes de Cotización)
  - Este objeto es una unidad de cada cotización hecha. Por practicidad y disposición de campos personalizados, decidí explotar al máximo los campos de tipo fórmula para automatizar cálculos sin necesidad de recurrir a código.
  En resumen, los campos que desarrollé fueron:
    - Fórmulas para obtención de porcentajes de becas aplicadas, así como el descuento de pago a contado.
    - Fórmula de total de descuentos que lo acota al 60%
    - Fórmula para obtención de Subtotal según el número de materias inscritas
    - Fórmula para obtención de Costo Total después de descuentos
    - Fórmula para dividir el Costo Total en mensualidades (4 meses si es Cuatrimestral, en 6 si es Semestral)

## Reglas de Validación
Dados los requerimientos de limitación de ciertos campos y los valores a introducir, los campos con validación fueron:
- Person Account: Promedio (Entre 0 y 10)
- Person Account: Beca de Excelencia (Promedio mayor de 9.5)
- Quote: Status (no poder avanzar en las fases de status sin que haya sido aprobada la cotización previamente)
- Opportunity Product: Quantity (Limitar el máximo de materias por periodo: 4 cuatrimestral, 7 semestral)

## Automatizaciones
### Approval Process: Quote Approval
- Condición de entrada: Status sea Needs Review o por medio del boton Submit for Approval
- Acción de entrada: Actualizar Status a In Review
- Acción final si fue aprobada: Actualizar Status a Approved y Reviewed? a True
- Acción final si fue rechazada: Actualizar Status a Rejected y Reviewed? a True
### Screen Flow: Generate Invoice PDF
  - Este Screen Flow fue insertado en la Record Page de Quote para facilitar su uso.
- Solicitar un email u ofrecer la opción de usar el email del account o solo visualizar la cotización a generar
- Si la Quote fue aprobada por el Approval Process, el PDF podrá ser generado, importado como documento mismo de la Quote y enviado a la dirección email señalada (ya sea input o del Email Account)
- Si la Quote no ha pasado por un Approval Process o fue rechazada, solo podrá ser previsualizada (activando el checkbox correspondiente)
- Si la Account no tiene email y se mandó a utilizar éste, o si la quote no cumple con los requisitos previamente mencionados, se mostrará una pantalla de error señalando el posible error.
- Se le envían 4 parámetros al controller extension de la página de Visualforce:
  - String: Quote Record ID
  - Boolean: Approved (Si la Quote ha sido aprobada o no)
  - String: Email (ya sea por input o el de la Account)
  - Boolean: Preview Only (si solo se busca previsualizar el pdf o no)
- Se recibe de vuelta al Flow un parámetro:
  - String: Preview URL (la dirección enlace para previsualizar el documento a generar)
- Por medio de una página de Visualforce con un Standard Controller a Quote y un Controller Extension, y gracias a los recursos recomendados por Liderly, se desarrolló la lógica donde:
  - Se reciben los 4 parámetros del Flow por medio de @invocableVariable's
  - Se hace un query a los Quote Line Items (if bulk) para crear adecuadamente la cotización con la orden correspondiente
  - En un @invocableMethod se reciben los parámetros y se llama la página de visualforce para, de cualquier manera, crear el documento
  - Se evalúa si solo será para previsualizar o, si es para enviar correo, se evalue si ha sido aprobado el Quote o no y se envía de vuelta al flow el URL para previsualizar el documento
  - Si es para enviar un correo:
    - Se genera el correo y se envía a la dirección email indicada
    - Se inserta a la Quote
## Extras
- Se renombaron las Tabs y Labels de los objetos utilizados para mejor comprensión (en idioma Español México)
- Se modificaron las Record Pages y los Page Layouts de los objetos para mejor entendimiento para la creación de records de cada uno de éstos
- Se creó una Plantilla de Quote para poder usar la acción 'Create PDF' del objeto Quote

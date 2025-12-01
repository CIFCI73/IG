## Introducción

Para esta práctica he decidido crear una simulación de una grúa pórtico industrial ("Gantry Crane") que organiza contenedores de colores de forma autónoma. He elegido este tema porque quería experimentar con el concepto de **jerarquía de objetos** (Scene Graph) y ver cómo coordinar movimientos complejos.

## Construcción de la Escena: La Jerarquía
La idea es que cada parte de la grúa es "hija" de la anterior. Así, si muevo la parte grande, las pequeñas le siguen automáticamente. Lo he organizado así en la función `construirGrua()`:

1.  **El Puente (Padre):** Es la estructura grande amarilla que se mueve adelante y atrás (Eje Z).
2.  **El Carro (Hijo):** Se mueve de izquierda a derecha sobre el puente (Eje X).
3.  **El Gancho (Nieto):** Sube y baja (Eje Y).

En el código se ve así:
```javascript
// El Puente es el grupo principal
puente = new THREE.Group();
scene.add(puente);

// ... (aquí creo las patas y la viga) ...

// El Carro es HIJO del puente
// el carro se mueve con él
puente.add(carro);

// El Gancho es HIJO del carro
carro.add(gancho);
```
Esto me facilitó muchísimo la vida. Por ejemplo, cuando quiero mover la grúa hacia una caja, solo tengo que decirle al Puente que vaya a z=5 y al Carro que vaya a x=3. No tengo que preocuparme de dónde está el gancho, porque sé que estará colgando debajo del carro.

También decidí crear las cajas de forma dinámica. En lugar de crear caja1, caja2, etc., he usado un array de objetos llamado lista_cajas donde defino qué tiene que hacer la grúa. Cada tarea tiene la posición de inicio, el destino y el color:
```javascript
var lista_cajas = [
  { start: { x: -6, z: 8 }, end: { x: 6, z: -5 }, color: 0xff0000 },
  { start: { x: -6, z: 3 }, end: { x: 6, z: 0 }, color: 0x00ff00 },
  { start: { x: -6, z: -2 }, end: { x: 6, z: 5 }, color: 0x0000ff },
  { start: { x: -6, z: -7 }, end: { x: 6, z: 10 }, color: 0xffff00 },
];
```
Luego, con un bucle for, recorro esta lista y llamo a crearCajas().

## La Lógica de Animación
La animación es una cadena de eventos para cada caja. Dentro de la función `animar()`, recorro mi lista de tareas y para cada una genero 6 tweens diferentes:
1.  **Viaje (Ir):** Mover el puente y el carro hacia la caja.
2.  **Bajar:** El gancho desciende hasta la altura de carga (-4.5).
3.  **Subir:** El gancho sube con la carga.
4.  **Transporte:** Llevar la caja a su destino final.
5.  **Descargar:** Bajar de nuevo.
6.  **Recuperar:** Subir el gancho vacío.

Para que esto se sienta real, he jugado bastante con las funciones de **Easing** (suavizado). Por ejemplo, para el movimiento de la grúa uso `Quadratic.InOut`, que hace que acelere al arrancar y frene suavemente al llegar, simulando la inercia de una máquina pesada de varias toneladas:
```javascript
var ir_puente = new TWEEN.Tween(puente.position)
  .to({ z: tarea.start.z }, 1500)
  .easing(TWEEN.Easing.Quadratic.InOut);
```
En cambio, para bajar el gancho uso Cubic.Out, que es más rápido al principio y frena mucho al final, para simular que el operario tiene cuidado de no golpear la carga contra el suelo.
```javascript
var bajar = new TWEEN.Tween(gancho.position)
      .to({ y: -5.3 }, 1000)
      .easing(TWEEN.Easing.Cubic.Out)
      .onComplete(function () {
        console.log("Caja cogida: " + index);
        agarre = true;
        cajaActual = tarea.mesh;
        cajaActual.material.emissive.setHex(0x333333);
      });
```














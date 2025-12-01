## Introducción

Para esta práctica he decidido crear una simulación de una grúa pórtico industrial ("Gantry Crane") que organiza contenedores de colores de forma autónoma. He elegido este tema porque quería experimentar con el concepto de **jerarquía de objetos** (Scene Graph) y ver cómo coordinar movimientos complejos.

## Construcción de la Escena: La Jerarquía
La idea es que cada parte de la grúa es "hija" de la anterior. Así, si muevo la parte grande, las pequeñas le siguen automáticamente. Lo he organizado así en la función `construirGrua()`:

1.  **El Puente (Padre):** Es la estructura grande amarilla que se mueve adelante y atrás.
2.  **El Carro (Hijo):** Se mueve de izquierda a derecha sobre el puente.
3.  **El Gancho (Nieto):** Sube y baja.

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
```javascript
function crearCajas() {
  var geo = new THREE.BoxGeometry(1.5, 1.5, 1.5);
  for (var i = 0; i < lista_cajas.length; i++) {
    var tarea = lista_cajas[i];
    var mat = new THREE.MeshStandardMaterial({ color: tarea.color });
    var caja = new THREE.Mesh(geo, mat);
    caja.position.set(tarea.start.x, 0.75, tarea.start.z);
    scene.add(caja);
    tarea.mesh = caja;
  }
}
```
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
Para que el movimiento del Puente (Eje Z) y el Carro (Eje X) empezaran exactamente al mismo tiempo, simplemente usé la función .onStart() en el primer tween para disparar el segundo:
```javascript
ir_puente.onStart(function() { ir_carro.start(); });
```
Finalmente, para que la grúa no intente mover todas las cajas a la vez, creé una variable llamada ultimoTween. Esta variable guarda la última animación del ciclo actual (subir el gancho vacío). Cuando el bucle pasa a la siguiente caja, le digo a esa última animación que active la primera de la nueva caja. De esta forma, se crea una cadena de animaciones generada automáticamente, sin importar cuántas cajas haya en la lista.

Implementé una lógica en el bucle de renderizado animationLoop para simular la física del agarre. Uso una variable simple agarre (true/false). Si es verdadera, actualizo la posición de la caja manualmente.

Aquí me encontré con uno de los problemas más difíciles. Al principio intenté simplemente igualar las posiciones (caja.position = gancho.position), pero la caja se iba a lugares extraños.
Me di cuenta de que esto pasaba por la jerarquía: la posición del gancho es local (relativa al carro, por ejemplo y=-2), pero la caja vive en el mundo global. Lo que hago es preguntar a Three.js la posición exacta del gancho en el mundo real usando `getWorldPosition()` y copio esos números a la caja:
```javascript
if (agarre && cajaActual) {
  var pos = new THREE.Vector3();
  gancho.getWorldPosition(pos); // Obtiene la posición global real
  
  cajaActual.position.x = pos.x;
  cajaActual.position.z = pos.z;
  cajaActual.position.y = pos.y - 1; // Un poco más abajo visualmente
}
```
Así consigo un agarre que parece magnético, sin que la caja se mueva durante el transporte.






---
title: Botones en la Polybard
published: true
---

![](/assets/images/general/Polybar.PNG)

La siguiente publicación te enseño como configurar botones en la polybard para así tener un acceso directo a las aplicaciones.

Lo primero es ubicarse en la ruta **~/.config/polybar.** Al estar en esa ruta se procede a abrir el archivo de configuración **current.ini**, cuando se tenga abierto nos ubicamos en la parte de modulos y veremos que viene con un modulo ya configurado para abrir **firefox**.

```js
[module/web]
  type = custom/text
  
  content = "%{T3}%{T-}"
  content-foreground = ${color.white}
  content-background = ${color.bg}
  content-padding = 0
  
  click-left  = firejail /opt/firefox/firefox &
```
Sino tienes agregado este modulo, se procede agregar. 

- Importante: 

	- El **click-left** es la acción que va a tomar el clic izquierdo a la hora de darle al icono, en este caso abrir firefox con firejail. (Puede agregarse cualquier acción)
	- El icono puede ser moficiado.

Cuando se tenga cofigurado el modulo se agrega en la parte superior la configuración visual del modulo agregado.

```js
[bar/firefox]
  inherit = bar/main
  width = 2%
  height = 40
  offset-x = 88.4%
  offset-y = 15
  background = ${color.bg}
  foreground = ${color.white}
  bottom = false
  padding = 0
  module-margin-left = 0
  module-margin-right = 0
  modules-center = web
  wm-restack = bspwm 
```

**Nota:** Es importante tener en cuenta que el **modules-center:web** se le pone el nombre del modulo creado anteriormente.

Cuando ya se tenga agregado el modulo y la configuración visual, se abre el archivo **launch.sh** ubicado en la ruta **~/.config/polybar** y se agrega la siguiente linea:

```js
polybar firefox -c ~/.config/polybar/current.ini &
```

**Nota:** El nombre despues de polyboard "firefox" es como se llama la configuración visual creada anteriormente.

Esta configuración aplica para todas las aplicaciones, lo importante es saber como es el comando para ejecutarse. 





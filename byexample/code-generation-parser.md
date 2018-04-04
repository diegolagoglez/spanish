# Generación de código (analizador sintáctico)

En este ejemplo se genera un analizador sintáctico de configuración en tiempo
de compilación. Pongamos como ejemplo que una aplicación tiene un par de
opciones de configuración, sintetizadas en una estructura:

```d
struct Config {
    int runs, port;
    string name;
}
```

Aunque escribir un analizador sintáctico para esta estructura no sería difícil,
tendríamos que estar constantemente actualizándolo cada vez que modificásemos
el objeto `Config`.

Es por esto por lo que estamos interesados en escribir una función `parse`
genérica que pueda leer opciones de configuración arbitrarias. Para simplificar,
la función `parse` aceptará un formato muy simple de opciones de configuración,
`clave1=valor1,clave2=valor2`, aunque se puede usar la misma técnica para
cualquier formato de configuración.

Para formatos de configuración más populares existen, evidentemente, lectores
en el [registro de DUB](https://code.dlang.org).

## Leyendo la configuración

Pongamos que el usuario tiene la siguiente cadena de caracteres como
configuración: `name=dlang,port=8080`. Directamente se separan las opciones
de configuración por la coma y se llama a la función `parse` con cada
elemento producto de esta separación. Después de que todas las opciones de
configuración hayan sido analizadas sintácticamente, se imprime toda la
configuración.

## Análisis sintáctico

Es en la función `parse` donde ocurre la magia, aunque antes de nada se
divide la cadena de configuración (por ejemplo *“name=dlang”*) por el
carácter *‘=’* en clave (*“name”*) y valor (*“dlang”*).

La sentencia `switch` se ejecuta con la clave analizada, pero la parte más
interesante es aquella en la que los casos de la sentencia `switch` se generan
de forma estática. La función `c.tupleof` devuelve una lista con todos los
miembros de la forma `(idx, name)`. El compilador detecta que `c.tupleof` se
conoce en tiempo de compilación con lo que el bucle `foreach` se desenrolla
también en tiempo de compilación.

La función `Conf.tupleof[idx].stringof` devuelve cada uno de los miembros
individuales de la estructura de la configuración, con lo que así se genera
una sentencia `case` dentro del `switch` para cada miembro de dicha estructura.

Igualmente, mientras se está dentro del bucle estático se puede acceder a cada
uno de los miembros de la estructura mediante su índice: `c.tupleof[idx]`; a
partir de ahí se puede asignar a cada miembro de la estructura el valor
analizado sintácticamente a partir de la cadena de configuración.

En este caso, además, se necesita la función `dropOne`, ya que el rango que
se ha divivido todavía apunta a la clave, con lo que `dropOne.front` devolverá
el segundo elemento, el valor.

Seguidamente, la función `to!T` es la que realiza la conversión real de la
cadena de entrada al correspondiente tipo de dato del miembro de la estructura
donde se asigna.

Finalmente, como el bucle `foreach` se desenrolla en tiempo de compilación,
es necesario un `break` para pararlo. Sin embargo, después de que una opción
de configuración ha sido correctamente analizada, no queremos saltar al
siguiente caso de la sentencia `switch`, por lo que se usa un `break` con
etiqueta para escapar de dicho `switch`.

## {SourceCode}

```d
import std.algorithm, std.conv, std.range,
        std.stdio, std.traits;

struct Config {
    int runs, port;
    string name;
}

void main() {
    Config conf;
    // Se usa el analizador generado
    // para cada elemento.
    "runs=1,port=2,name=hello"
        .splitter(",")
        .each!(e => conf.parse(e));
    conf.writeln;
}

void parse(Conf)(ref Conf c, string entry) {
    auto r = entry.splitter("=");

    Switch:
    switch (r.front) {
        foreach (idx, _; c.tupleof) {
            alias T = typeof(Conf.tupleof[idx]);
            case Conf.tupleof[idx].stringof:
                c.tupleof[idx] = r.dropOne
                                  .front.to!T;
                break Switch;
        }
        default:
            assert (0, "Miembro desconocido.");
    }
}
```

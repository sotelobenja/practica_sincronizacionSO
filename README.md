# Caso práctico: sincronización de relojes en sistemas distribuidos

Implementación en C# / ASP.NET Core y Docker Compose de los tres ejercicios del PDF, siguiendo las guías `.md` de la carpeta.

| Carpeta | Algoritmo | Cómo ejecutar |
|---|---|---|
| `CristianSync/` | Cristian (servidor de tiempo + 3 clientes) | `cd CristianSync && docker compose up --build` |
| `BerkeleyLab/` | Berkeley (coordinador + 3 nodos) | `cd BerkeleyLab && docker compose up -d --build` |
| `LamportPedidos/` | Lamport (Orders, Inventory, Payments) | `cd LamportPedidos && docker compose up -d --build` |

No conviene levantar Berkeley y Lamport al mismo tiempo: ambos publican los puertos 8081–8083.

El informe con respuestas, tablas y comparación está en `INFORME.md`.

Los ajustes de Cristian y Berkeley se aplican **solo al reloj simulado de la aplicación**, nunca a la hora del sistema operativo.

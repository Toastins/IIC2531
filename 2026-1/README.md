# IIC2531 - Seguridad Computacional

## Información General

| | |
|---|---|
| **Código** | IIC2531 |
| **Semestre** | 2026-1 |
| **Créditos** | 10 |
| **Horario** | Lunes y Miércoles, 8:20 - 9:40 hrs |
| **Sala** | A4 |
| **Profesor** | Ignacio Parada (yo@ignacioparada.com) |

## Descripción

Este curso ofrece una visión general de los principales temas en seguridad computacional. A través de lecturas de papers académicos y de la industria, los estudiantes explorarán técnicas de ataque y defensa en sistemas reales. El enfoque es práctico: los laboratorios permiten experimentar con vulnerabilidades y contramedidas en el contexto de una aplicación web.

El curso prioriza **amplitud sobre profundidad**, cubriendo una variedad de temas que van desde aislamiento de procesos hasta seguridad de redes, pasando por arquitecturas de sistemas seguros y defensa de software.

## Contenidos

### Módulo 1: Aislamiento
- **Modelos de amenaza:** identificación de adversarios, superficies de ataque, y objetivos de seguridad
- **Procesos y separación de privilegios:** principio de mínimo privilegio, diseño de OKWS
- **Contenedores Linux:** namespaces (PID, net, mount), cgroups, chroot, seccomp-bpf
- **Máquinas virtuales:** KVM, QEMU, virtualización de CPU/memoria/dispositivos
- **Sandboxing moderno:** Firecracker, microVMs, trade-offs entre aislamiento y rendimiento

### Módulo 2: Casos de Estudio en Arquitecturas de Seguridad
- **Arquitecturas de aplicaciones web:** separación de componentes, servicios de autenticación
- **Sistemas operativos seguros:** capacidades, control de acceso mandatorio (SELinux, AppArmor)
- **Aislamiento en navegadores:** modelo de seguridad de origen, sandboxing de procesos
- **Sistemas distribuidos:** coordinación segura, consenso, manejo de fallas bizantinas

### Módulo 3: Seguridad de Software
- **Vulnerabilidades de memoria:** buffer overflow, use-after-free, format strings
- **Técnicas de explotación:** inyección de código, ROP (Return-Oriented Programming)
- **Defensas en tiempo de ejecución:** stack canaries, ASLR, DEP/NX, CFI
- **Análisis de código:** fuzzing, análisis estático, ejecución simbólica
- **Seguridad en la cadena de suministro:** dependencias, reproducibilidad, firmas de código

### Módulo 4: Seguridad en Redes y Sistemas Distribuidos
- **Criptografía aplicada:** cifrado simétrico y asimétrico, hashing, firmas digitales
- **Protocolos seguros:** TLS/HTTPS, certificados X.509, Certificate Transparency
- **Autenticación:** passwords, hashing con salt (PBKDF2), tokens, WebAuthn/FIDO2
- **Seguridad web:** mismo origen, CORS, cookies, CSRF, XSS, inyección SQL

## Evaluación

La nota final se calcula de la siguiente manera:

```
NotaFinal = 0.4 × Evaluaciones + 0.4 × Laboratorios + 0.2 × Proyecto
```

Donde:
- **Evaluaciones** = 0.3 × I₁ + 0.3 × I₂ + 0.4 × Examen
- **Laboratorios** = 0.1 × L₁ + 0.2 × L₂ + 0.3 × L₃ + 0.3 × L₄ + 0.1 × L₅

### Requisitos de Aprobación
- Evaluaciones ≥ 4.0
- Examen ≥ 4.0
- Proyecto ≥ 4.0

Si alguno de estos requisitos no se cumple, la nota final corresponde a la nota del componente reprobado.

### Fechas de Evaluaciones

| Evaluación | Fecha |
|------------|-------|
| I1 | 27 de Abril, 2026 |
| I2 | 13 de Junio, 2026 |
| Examen | 9 de Julio, 2026 |

### Fechas de Laboratorios

| Lab | Tema | Fecha de Entrega |
|-----|------|------------------|
| Lab 1 | Modelo de Amenaza | 18 de Marzo, 2026 |
| Lab 2 | Separación de Privilegios | 8 de Abril, 2026 |
| Lab 3 | Buffer Overflow | 4 de Mayo, 2026 |
| Lab 4 | Seguridad en Navegadores | 25 de Mayo, 2026 |
| Lab 5 | Por definir | 8 de Junio, 2026 |

### Proyecto Final
- **Fecha de entrega:** 22 de Junio, 2026
- Puede ser orientado a ataque o defensa
- Presentaciones al final del semestre

## Bibliografía

### Textos de Referencia
- Anderson, R. *Security Engineering: A Guide to Building Dependable Distributed Systems* (3rd ed.). Wiley, 2020. [Disponible online](https://www.cl.cam.ac.uk/~rja14/book.html)
- Viega, J. & McGraw, G. *Building Secure Software*. Addison-Wesley, 2001.

### Papers y Lecturas
El curso se basa en papers académicos y de la industria que se asignan clase a clase. Los estudiantes deben leer el material asignado antes de cada sesión.

---

*Este curso está basado en materiales de MIT 6.858 Computer Systems Security, utilizado bajo licencia [Creative Commons Attribution 3.0](http://creativecommons.org/licenses/by/3.0/us/).*

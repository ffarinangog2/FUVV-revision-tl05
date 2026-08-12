# FORM-TL-05 — Revisión Técnica Cruzada · Equipo FUVV

Informe de revisión del PFC del equipo **ACC — Soporte Técnico ISP**, realizado por el equipo
**FUVV** para el taller FORM-TL-05 de Aplicaciones Distribuidas (ISR-701), Unidad 5.

## Equipo revisor (FUVV)

| Integrante | Bloques asignados |
|---|---|
| Freddy Vladimir Farinango Guandinango | A (SOLID) y B (patrones) |
| Isaías Abraham Urbina Romero | C (capas), D (excepciones) y E (observabilidad) |

## Objeto revisado

- **Repositorio:** https://github.com/carlospatroner-boop/PFC-SOPORTE-ISP
- **Commit auditado:** `479b637` (`479b6370a640c0d2edfad801e6937e9ede8c3c80`)
- **Rama:** `feature/entrega-3`
- **Archivos versionados:** 1795
- **Fecha de auditoría:** martes 11 de agosto de 2026

## Contenido del repositorio

```
informe_revision.tex   Archivo principal (fuente LaTeX)
informe_revision.pdf   PDF compilado
issues.md              Cuerpo de los 12 issues publicados en el repositorio revisado
README.md              Este archivo
```

## Instrucciones de compilación

**Compilador:** pdfLaTeX
**Archivo principal:** `informe_revision.tex`

```bash
pdflatex informe_revision.tex
pdflatex informe_revision.tex
pdflatex informe_revision.tex
```

Se ejecuta tres veces para resolver el índice, las referencias cruzadas de tablas y figuras y
la paginación de las tablas largas (`xltabular`).

**No se requiere `biber` ni `bibtex`.** La bibliografía se declara con el entorno
`thebibliography` en formato IEEE dentro del propio `.tex`, de modo que el PDF se genera desde
un clon limpio con pdfLaTeX únicamente.

### Dependencias

Distribución TeX completa (TeX Live o MiKTeX) con los paquetes:

`geometry`, `booktabs`, `tabularx`, `xltabular`, `longtable`, `array`, `caption`,
`enumitem`, `listings`, `xcolor`, `tcolorbox`, `xurl`, `hyperref`, `inputenc`, `fontenc`.

Todos pertenecen a la instalación estándar de TeX Live. No se usan imágenes externas: el mapa
de arquitectura se compone dentro del documento, por lo que no existen rutas locales que
impidan la compilación desde un clon limpio.

### Compilación verificada

Última compilación desde clon limpio: **11 páginas, 0 errores, 0 desbordes de margen.**

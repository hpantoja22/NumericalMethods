# Caso de Estudio: La Tubería que no Convergía

**Curso:** Métodos Numéricos · **Autor:** Dr. Hermes Pantoja Carhuavilca
**Tema:** Ecuaciones no lineales — Ecuación de Colebrook *(caso ilustrativo)*

---

## 1. Contexto

Se debe seleccionar la bomba de una línea de agua de acero comercial con los siguientes datos:

| Parámetro | Valor |
|---|---|
| Diámetro $D$ | 0.30 m |
| Longitud $L$ | 1000 m |
| Velocidad $V$ | 2 m/s |
| Rugosidad $\varepsilon$ | 0.045 mm |
| Viscosidad cinemática $\nu$ | $10^{-6}$ m²/s |
| Número de Reynolds | $\mathrm{Re} = VD/\nu = 6\times10^{5}$ |

El factor de fricción $f$ es la raíz de la **ecuación de Colebrook** (no lineal e implícita):

$$
F(f)=\frac{1}{\sqrt f}+2\log_{10}\!\left(\frac{\varepsilon/D}{3.7}+\frac{2.51}{\mathrm{Re}\sqrt f}\right)=0
$$

## 2. El fallo técnico

Un script aplica **Newton–Raphson directamente sobre $f$**, con el valor inicial "típico" $f_0 = 0.05$:

$$
f_{1}=f_0-\frac{F(f_0)}{F'(f_0)} = 0.05-\frac{-3.9825}{-47.464} = -0.0339
$$

La primera iteración **sale del dominio físico** ($f<0$ ⇒ $\sqrt f$ de un número negativo).

## 3. La consecuencia

MATLAB **no se detiene**: continúa operando con números complejos y las iteraciones explotan:

| $k$ | $f_k$ |
|---|---|
| 1 | $-0.0339$ |
| 2 | $-0.0989 + 0.1036i$ |
| 3 | $0.5367 + 0.6095i$ |
| 4 | $-1.908 - 9.529i$ |
| 5 | $425.8 + 202.1i$ |
| 6 | $-1.29\times10^{5} - 1.01\times10^{5}i$ |

> ⚠️ Un programa que toma `real(f)` reporta un resultado sin sentido **sin ningún aviso**.

---

## 4. Diagnóstico: ¿qué salió mal?

### Fallo 1 — Newton sin proteger el dominio

| $f_0$ | Resultado de Newton en $f$ |
|---|---|
| 0.02 | ✅ Converge a 0.014695 (4 iteraciones) |
| 0.05 | ❌ $f_1 = -0.0339$ |
| 0.10 | ❌ $f_1 = -0.2185$ |
| 1.00 | ❌ $f_1 = -13.24$ |

Newton es un **método abierto**: sin un buen valor inicial ni protección del dominio puede salir de la región física o diverger.

### Fallo 2 — Criterio de parada laxo

Detener la bisección cuando $|F(f)| < 0.5$ entrega $f = 0.01338$ en solo 3 iteraciones. La pérdida de carga es $h_f = f\,\dfrac{L}{D}\,\dfrac{V^2}{2g}$:

| Caso | $f$ | $h_f$ (m) |
|---|---|---|
| Correcto | 0.014695 | 9.99 |
| Criterio laxo | 0.013375 | **9.09** |

> ❌ La pérdida de carga se **subestima un 9 %** ⇒ bomba subdimensionada.

---

## 5. Estrategias robustas

1. **Cambio de variable** $x = 1/\sqrt f$:
   $$
   G(x)=x+2\log_{10}\!\left(\frac{\varepsilon/D}{3.7}+\frac{2.51\,x}{\mathrm{Re}}\right)=0
   $$
   La función es casi lineal: Newton converge desde **cualquier** $x_0\in[1,50]$ en 3–4 iteraciones.
2. **Acotar primero** (Teorema de Bolzano / bisección) y luego refinar con Newton.
3. **Punto fijo** $f_{k+1}=\left[-2\log_{10}\!\left(\dfrac{\varepsilon/D}{3.7}+\dfrac{2.51}{\mathrm{Re}\sqrt{f_k}}\right)\right]^{-2}$: converge en 5 iteraciones.
4. **Criterio de parada doble:** $|\varepsilon_a| < \varepsilon_s$ **y** `maxIter`, verificando $f>0$ en cada paso.

**Resultado validado:** $f = 0.014695$, $h_f = 9.99$ m, $\Delta p \approx 98$ kPa.

---

## 6. Implementación robusta en MATLAB

```matlab
D = 0.30; e = 0.045e-3; Re = 6e5;
a = (e/D)/3.7;  b = 2.51/Re;
G  = @(x) x + 2*log10(a + b*x);           % x = 1/sqrt(f)
dG = @(x) 1 + (2/log(10))*b./(a + b*x);

x = 5; tol = 1e-8; maxIter = 50;
for k = 1:maxIter
    xn = x - G(x)/dG(x);
    if xn <= 0 || ~isreal(xn)
        error('Fuera del dominio fisico');
    end
    ea = abs((xn - x)/xn);
    x  = xn;
    if ea < tol, break; end
end
f = 1/x^2      % f = 0.014695 en 4 iteraciones
```

### Checklist de ingeniería

- [ ] ¿La formulación es **bien comportada** (casi lineal) en la variable elegida?
- [ ] ¿Se verifica que cada iterado permanezca en el **dominio físico**?
- [ ] ¿Se usa $|\varepsilon_a|$ con tolerancia estricta **y** límite de iteraciones?
- [ ] ¿Se **valida** el resultado (sustituir en $F$, comparar con el diagrama de Moody)?

---

## 7. La lección: converger no es lo mismo que acertar

**Análisis del fallo**

- **Error:** no era un *bug* de sintaxis; Newton hizo exactamente lo que dice su fórmula.
- **Causa raíz:** se ignoró que Newton es un método **abierto**: sin un buen $x_0$ ni protección del dominio, puede salir de la región física o diverger.
- **El costo:** una tolerancia mal elegida produjo un resultado "convergido" que subestima la pérdida de carga en un 9 %.

> *En ecuaciones no lineales, la elección de la **formulación**, el **valor inicial** y el **criterio de parada** es una responsabilidad del ingeniero: los métodos cerrados aportan seguridad, los abiertos aportan velocidad, y la combinación de ambos aporta confianza.*

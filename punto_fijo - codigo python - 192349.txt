import tkinter as tk
from tkinter import ttk, messagebox
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import math
import re

# ──────────────────────────────────────────────
#  COLORES Y ESTILO
# ──────────────────────────────────────────────
BG        = "#0f1117"
PANEL     = "#1a1d27"
ACCENT    = "#00d4aa"
ACCENT2   = "#ff6b6b"
TEXT      = "#e8eaf0"
SUBTEXT   = "#8b8fa8"
ENTRY_BG  = "#252836"
BORDER    = "#2e3245"
SUCCESS   = "#4caf90"
WARNING   = "#f0a500"

FONT_TITLE  = ("Courier New", 18, "bold")
FONT_LABEL  = ("Courier New", 10)
FONT_MONO   = ("Courier New", 10)
FONT_RESULT = ("Courier New", 12, "bold")


# ──────────────────────────────────────────────
#  PREPROCESADOR DE EXPRESIONES
#  Convierte notación matemática común a sintaxis Python/math
# ──────────────────────────────────────────────
def preprocesar(expr):
    expr = expr.strip()
    expr = re.sub(r'\bsen\b', 'sin', expr)       # sen → sin (español)
    expr = re.sub(r'\btg\b',  'tan', expr)       # tg  → tan (español)
    expr = re.sub(r'\bln\b',  'log', expr)       # ln  → log (logaritmo natural)
    expr = re.sub(r'\be\s*\^\s*\(', 'exp(', expr)              # e^( → exp(
    expr = re.sub(r'\be\s*\^\s*(-?[a-zA-Z0-9_.]+)', r'exp(\1)', expr)  # e^x → exp(x)
    expr = re.sub(r'\^', '**', expr)             # ^ → ** (potencia en Python)
    return expr


# ──────────────────────────────────────────────
#  ENTORNO DE EVALUACIÓN SEGURO
#  Crea un namespace con funciones matemáticas permitidas
# ──────────────────────────────────────────────
def entorno_math():
    env = {k: getattr(math, k) for k in dir(math) if not k.startswith("_")}
    env["abs"] = abs
    return env


# ──────────────────────────────────────────────
#  MÉTODO DE PUNTO FIJO
#  Itera xₙ₊₁ = g(xₙ) hasta que el error relativo sea ≤ tolerancia
# ──────────────────────────────────────────────
# NOTA:
# El criterio de parada usa error relativo entre iteraciones.
# Esto no garantiza que f(x) = 0, solo que la secuencia converge.
def punto_fijo(g_expr, x0, tol, max_iter=100):
    g_expr = preprocesar(g_expr)
    allowed = entorno_math()

    def g(x):
        local = dict(allowed)
        local["x"] = x
        return eval(g_expr, {"__builtins__": {}}, local)

    iteraciones = []
    x = x0

    for i in range(1, max_iter + 1):
        try:
            gx = g(x)
        except Exception as e:
            raise ValueError(f"Error al evaluar g(x) en x={x:.6f}: {e}")

        # Redondear a 4 decimales para evitar acumulación de error de punto flotante
        gx = round(gx, 4)
        x  = round(x, 4)

        # Error relativo porcentual: |g(xₙ) - xₙ| / |g(xₙ)|
        error = abs((gx - x) / gx) if gx != 0 else abs(gx - x)
        iteraciones.append({"n": i, "xn": x, "gxn": gx, "error": error})

        # Criterio de parada: el error ya es suficientemente pequeño
        if error <= tol:
            break
        x = gx

    return iteraciones


# ──────────────────────────────────────────────
#  GENERADOR AUTOMÁTICO DE CANDIDATAS g(x)
#  A partir de f(x) = 0 produce varias reescrituras x = g(x)
#  usando álgebra simbólica básica sobre la expresión de texto.
# ──────────────────────────────────────────────
# NOTA:
# No todas las transformaciones g(x) garantizan convergencia.
# La condición ideal es |g'(x)| < 1 cerca de la raíz.
def generar_candidatas_gx(f_expr_raw, x0):
    """
    Recibe f(x) como texto y devuelve una lista de tuplas
        (descripcion, g_expr_str)
    con candidatas a g(x) que se pueden intentar para punto fijo.

    Estrategias implementadas:
      1. g(x) = x - f(x)            → desplazamiento directo
      2. g(x) = x - f(x)/c          → escalado con c ∈ {2, 5, 10}
      3. g(x) = x + f(x)/c          → escalado con signo positivo
      4. g(x) = x - α·f(x)          → relajación con α pequeño
      5. Variantes numéricas de derivada aproximada (Newton-like)
    """
    f = preprocesar(f_expr_raw)

    candidatas = []

    # ── Estrategia 1: g(x) = x - f(x) ──────────
    candidatas.append(("x - f(x)", f"x - ({f})"))

    # ── Estrategia 2/3: escalado ─────────────────
    for c in [2, 3, 5, 10]:
        candidatas.append((f"x - f(x)/{c}", f"x - ({f})/{c}"))
        candidatas.append((f"x + f(x)/{c}", f"x + ({f})/{c}"))

    # ── Estrategia 4: relajación α pequeño ───────
    for alpha in [0.1, 0.01, 0.5]:
        candidatas.append((f"x - {alpha}·f(x)", f"x - {alpha}*({f})"))

    # ── Estrategia 5: Newton-like con h pequeño ──
    # g(x) = x - f(x)/f'(x)  aproximando f'(x) ≈ (f(x+h)-f(x))/h
    candidatas.append((
        "x - f(x)/f'(x)  [Newton aprox.]",
        f"x - ({f}) / (({f.replace('x', '(x+1e-5)')}) - ({f})) * 1e-5"
    ))

    return candidatas


# ──────────────────────────────────────────────
#  DETECTOR DE DISCONTINUIDADES / ASÍNTOTAS
#  Evalúa f cerca de x para detectar valores explosivos o no finitos
#  que indican una asíntota (ejemplo: tan(x) cerca de π/2, 3π/2...).
# ──────────────────────────────────────────────
def cerca_de_asintota(f_expr, x, umbral=1e6):
    """
    Evalúa f_expr en x y en puntos ligeramente desplazados.
    Si algún valor absoluto supera `umbral` o no es finito,
    asume que x está cerca de una discontinuidad vertical.

    Parámetros:
        f_expr : expresión de f(x) ya preprocesada con preprocesar()
        x      : punto a evaluar
        umbral : magnitud a partir de la cual se sospecha asíntota

    Retorna True si hay riesgo de discontinuidad, False si el punto es seguro.
    """
    allowed = entorno_math()
    errores = 0

    for d in [0, 1e-6, -1e-6]:
        try:
            local = dict(allowed)
            local["x"] = x + d
            val = eval(f_expr, {"__builtins__": {}}, local)

            if not math.isfinite(val) or abs(val) > umbral:
                errores += 1

        except Exception:
            errores += 1

    # Solo marcar como peligrosa si hay múltiples fallos
    return errores >= 2


# ──────────────────────────────────────────────
#  DETECTOR DE OSCILACIÓN / DIVERGENCIA EXPLOSIVA
#  Compara el error promedio de la primera y segunda mitad de una
#  ventana de iteraciones recientes para identificar tendencias crecientes.
# ──────────────────────────────────────────────
def detectar_divergencia(historial, ventana=6, factor=8):
    """
    Examina las últimas `ventana` iteraciones del historial.
    Si el error promedio de la segunda mitad es `factor` veces mayor
    al de la primera mitad, clasifica la secuencia como divergente.

    Parámetros:
        historial : lista de dicts con al menos la clave 'error'
        ventana   : iteraciones recientes a considerar
        factor    : razón de crecimiento que se considera explosiva

    Retorna True si se detecta divergencia, False si la tendencia es estable.
    """
    if len(historial) < ventana:
        return False  # Sin suficientes datos todavía

    recientes = historial[-ventana:]
    mitad     = ventana // 2
    primera   = [it["error"] for it in recientes[:mitad]]
    segunda   = [it["error"] for it in recientes[mitad:]]

    prom1 = sum(primera) / len(primera) if primera else 0
    prom2 = sum(segunda) / len(segunda) if segunda else 0

    # Divergencia sostenida: la segunda mitad de la ventana es `factor` veces peor
    if prom1 > 0 and (prom2 / prom1) > factor:
        return True

    # Divergencia inmediata: error reciente absurdamente alto (>50 = 5000%)
    if any(it["error"] > 50 for it in recientes[-2:]):
        return True

    return False


# ──────────────────────────────────────────────
#  VERIFICADOR DE CONVERGENCIA
#  Prueba cada g(x) candidata y devuelve la primera que converge.
#
#  Mejoras implementadas:
#    · cerca_de_asintota(): filtra x0 que caigan en zonas peligrosas de f
#    · Filtro de salto abrupto entre iteraciones consecutivas (|Δx| > 1e5)
#    · detectar_divergencia(): abandona candidatas con error explosivo
#    · Todos los filtros están comentados indicando qué caso cubren
# ──────────────────────────────────────────────
def buscar_gx_convergente(f_expr, x0, tol, max_iter=50):
    """
    Intenta las candidatas de g(x) generadas a partir de f(x).
    Una candidata se considera convergente si:
      - x0 no produce valores explosivos al evaluar f (sin asíntotas cercanas)
      - No lanza excepciones durante las iteraciones
      - Los valores xₙ permanecen acotados (|xₙ| < 1e8)
      - No hay saltos abruptos entre pasos consecutivos (|Δx| < 1e5)
      - No se detecta divergencia explosiva sostenida
      - El error relativo cae por debajo de la tolerancia ε

    Parámetros:
        f_expr   : función f(x) ingresada por el usuario (sin preprocesar)
        x0       : punto inicial de la iteración
        tol      : tolerancia de parada (error relativo)
        max_iter : máximo de iteraciones de prueba por candidata

    Retorna (descripcion, g_expr, iteraciones) de la primera candidata
    que converge, o None si ninguna supera todos los filtros.
    """
    candidatas = generar_candidatas_gx(f_expr, x0)
    allowed    = entorno_math()
    f_prepro   = preprocesar(f_expr)  # Versión preprocesada para chequeos auxiliares

    for desc, g_expr_raw in candidatas:
        g_expr = preprocesar(g_expr_raw)

        def hacer_g(expr):
            """Cierre que captura `expr` por valor para evitar la trampa del bucle."""
            def g(x):
                local = dict(allowed)
                local["x"] = x
                return eval(expr, {"__builtins__": {}}, local)
            return g

        g = hacer_g(g_expr)

        try:
            x            = x0
            convergio    = False
            iters_prueba = []

            for i in range(1, max_iter + 1):
                gx = g(x)

                # ── Filtro 1: NaN / Inf / desbordamiento ────────────────────
                # Cubre asíntotas verticales (tan cerca de π/2) y overflow.
                if not math.isfinite(gx) or abs(gx) > 1e8:
                    break

                # ── Filtro 2: salto abrupto entre pasos consecutivos ─────────
                # Si la iteración cruza una asíntota el valor cambia en
                # varios órdenes de magnitud de un paso al siguiente.
                if iters_prueba and abs(gx - iters_prueba[-1]["gxn"]) > 1e5:
                    break

                gx_r  = round(gx, 4)
                x_r   = round(x,  4)
                error = abs((gx_r - x_r) / gx_r) if gx_r != 0 else abs(gx_r - x_r)
                iters_prueba.append({"n": i, "xn": x_r, "gxn": gx_r, "error": error})

                # ── Criterio de convergencia ─────────────────────────────────
                if error <= tol:
                    convergio = True
                    break

                # ── Filtro 3: divergencia sostenida en ventana reciente ───────
                # Detecta cuando el error crece de forma explosiva y sostenida
                # (no solo un pico puntual), lo que indica que la candidata
                # no converge aunque eventualmente pueda estabilizarse por azar.
                if detectar_divergencia(iters_prueba):
                    break

                x = gx

            if convergio:
                # Recalcular las iteraciones completas con la g(x) ganadora
                # para garantizar coherencia total en tabla y gráficas.
                iters_completas = punto_fijo(g_expr_raw, x0, tol)
                return desc, g_expr_raw, iters_completas

        except Exception:
            # Error matemático en esta candidata (dominio, div/0, etc.)
            # No es fatal: simplemente se descarta y se prueba la siguiente.
            continue

    return None  # Ninguna candidata superó todos los filtros


# ──────────────────────────────────────────────
#  VENTANA DE MANUAL DE INSTRUCCIONES
# ──────────────────────────────────────────────
MANUAL_SECCIONES = [
    {
        "titulo": "¿Qué es el Método de Punto Fijo?",
        "icono": "◈",
        "color": ACCENT,
        "contenido": (
            "El método de punto fijo es una técnica iterativa para encontrar raíces de ecuaciones. "
            "Dada una ecuación f(x) = 0, se reescribe en la forma x = g(x). Luego se itera:\n\n"
            "        xₙ₊₁ = g(xₙ)\n\n"
            "partiendo de un valor inicial x₀, hasta que la diferencia entre iteraciones "
            "sea menor que la tolerancia ε definida."
        )
    },
    {
        "titulo": "Cómo usar la aplicación (nueva versión)",
        "icono": "◉",
        "color": ACCENT,
        "contenido": (
            "1.  Ingresa f(x) en el campo 'Función f(x)'.\n"
            "        Ejemplo:  x**2 - x - 2\n\n"
            "2.  Ingresa el valor inicial x₀ (punto de partida de la búsqueda).\n"
            "        Ejemplo:  1.5\n\n"
            "3.  Ingresa la tolerancia ε (criterio de parada).\n"
            "        Ejemplo:  0.0001\n\n"
            "4.  Presiona  ▶ BUSCAR g(x) Y CALCULAR.\n"
            "        El programa probará varias reescrituras x = g(x) automáticamente\n"
            "        y usará la primera que converja.\n\n"
            "5.  El campo 'g(x) encontrada' mostrará cuál g(x) fue seleccionada.\n\n"
            "6.  Presiona  ✕ LIMPIAR  para restablecer todos los campos."
        )
    },
    {
        "titulo": "Cómo escribir la función f(x)",
        "icono": "◆",
        "color": WARNING,
        "contenido": (
            "Usa sintaxis Python estándar. La variable debe llamarse  x  (minúscula).\n\n"
            "  Potencias:      x**2   o   x^2       →  x²\n"
            "  Raíz cuadrada:  sqrt(x)\n"
            "  Exponencial:    exp(x)  o  e^x        →  eˣ\n"
            "  Logaritmo nat.: log(x)  o  ln(x)\n"
            "  Seno/Coseno:    sin(x)  /  cos(x)     (también: sen, tg)\n"
            "  Constantes:     pi  →  3.14159...  |  e  →  2.71828...\n\n"
            "Ejemplos válidos de f(x):\n"
            "  x**2 - x - 2\n"
            "  cos(x) - x\n"
            "  exp(-x) - x\n"
            "  x**3 - 2*x - 5\n"
            "  log(x) - cos(x)"
        )
    },
    {
        "titulo": "Interpretación de resultados",
        "icono": "◎",
        "color": SUCCESS,
        "contenido": (
            "g(x) ENCONTRADA\n"
            "  Muestra la reescritura x = g(x) que el programa eligió automáticamente.\n\n"
            "TABLA DE ITERACIONES\n"
            "  n          →  Número de iteración\n"
            "  xₙ         →  Valor de x en la iteración actual\n"
            "  g(xₙ)      →  Valor calculado g(xₙ) = siguiente x\n"
            "  Error (%)  →  Error porcentual relativo: |g(xₙ)−xₙ| / |g(xₙ)| × 100\n"
            "  Fila verde →  Última iteración (donde se cumplió la tolerancia)\n\n"
            "RESULTADO FINAL\n"
            "  Verde  →  El método convergió: se encontró una raíz aproximada x*\n"
            "  Naranja →  No convergió (ninguna g(x) funcionó)\n\n"
            "GRÁFICAS\n"
            "  Superior →  Evolución de xₙ hacia la raíz x*\n"
            "  Inferior →  Decrecimiento del error por iteración"
        )
    },
    {
        "titulo": "Condición de convergencia",
        "icono": "◇",
        "color": ACCENT2,
        "contenido": (
            "El método converge si |g'(x)| < 1 cerca de la raíz.\n\n"
            "El programa prueba automáticamente estas estrategias de reescritura:\n\n"
            "  • g(x) = x − f(x)\n"
            "  • g(x) = x − f(x)/c   para c ∈ {2, 3, 5, 10}\n"
            "  • g(x) = x + f(x)/c   (variante de signo)\n"
            "  • g(x) = x − α·f(x)  para α ∈ {0.1, 0.01, 0.5}\n"
            "  • Aproximación tipo Newton\n\n"
            "Si ninguna converge, intenta cambiar x₀ más cerca de la raíz."
        )
    },
    {
        "titulo": "Errores comunes",
        "icono": "⚠",
        "color": ACCENT2,
        "contenido": (
            "• 'No se encontró g(x) convergente'\n"
            "  →  Prueba un x₀ más cercano a la raíz esperada.\n\n"
            "• 'math domain error'\n"
            "  →  La función no está definida en ese dominio (ej: sqrt de negativo,\n"
            "     log de cero). Cambia x₀.\n\n"
            "• Resultado muy diferente al esperado\n"
            "  →  Revisa los paréntesis. Ej:  x+2/3  ≠  (x+2)/3\n\n"
            "• La raíz encontrada no es la que buscabas\n"
            "  →  Cambia x₀ para dirigir el método hacia otra raíz."
        )
    },
    {
        "titulo": "Funciones con discontinuidades (tan, cot, sec...)",
        "icono": "⚡",
        "color": WARNING,
        "contenido": (
            "Funciones como tan(x), cot(x) o 1/sin(x) tienen asíntotas verticales\n"
            "donde el valor se dispara a ±∞. Si x₀ cae cerca de esos puntos,\n"
            "el método puede oscilar o divergir de forma caótica.\n\n"
            "Asíntotas de tan(x):  x = π/2 ≈ 1.5708,  3π/2 ≈ 4.7124,  5π/2 ≈ 7.8540...\n\n"
            "El programa detecta automáticamente estos casos y:\n"
            "  · Descarta candidatas que crucen una asíntota durante la iteración.\n"
            "  · Muestra una advertencia específica si x₀ está en zona peligrosa.\n\n"
            "Solución: elige x₀ en el mismo 'tramo continuo' que la raíz buscada.\n"
            "  Ej: para tan(x) − x con raíz en x ≈ 0, usa x₀ = 0.1 (no 4.5).\n"
            "  Ej: para la raíz en x ≈ 7.79, usa x₀ = 7.5 (entre 3π/2 y 5π/2)."
        )
    },
    {
        "titulo": "Aclaración del criterio de parada",
        "icono": "⚠",
        "color": ACCENT2,
        "contenido": (
            "El criterio de finalización se basa en el error relativo entre iteraciones:\n\n"
            "        ε = |xₙ₊₁ − xₙ| / |xₙ₊₁|\n\n"
            "Cuando ε es menor que la tolerancia, el método se detiene.\n\n"
            "IMPORTANTE:\n"
            "Un error pequeño indica que las iteraciones se han estabilizado,\n"
            "pero NO garantiza que el valor encontrado sea una raíz de f(x).\n\n"
            "En algunos casos, el método puede converger a un punto fijo de g(x)\n"
            "que no cumple f(x) = 0.\n\n"
            "Por ello, es recomendable verificar el resultado evaluando f(xₙ)."
        )
    }
]


class VentanaManual:
    """Ventana secundaria que muestra el manual de instrucciones con scroll."""
    def __init__(self, parent):
        self.win = tk.Toplevel(parent)
        self.win.title("Manual de Instrucciones")
        self.win.configure(bg=BG)
        self.win.geometry("640x680")
        self.win.resizable(True, True)
        self.win.transient(parent)  # Asociada a la ventana principal
        self._build()

    def _build(self):
        # ── Encabezado ────────────────────────
        head = tk.Frame(self.win, bg=PANEL, pady=16)
        head.pack(fill="x")
        tk.Label(head, text="◈  MANUAL DE INSTRUCCIONES",
                        font=("Courier New", 14, "bold"), fg=ACCENT, bg=PANEL).pack(side="left", padx=20)
        tk.Label(head, text="Método de Punto Fijo",
                    font=("Courier New", 9), fg=SUBTEXT, bg=PANEL).pack(side="left")

        tk.Frame(self.win, height=2, bg=BORDER).pack(fill="x")

        # ── Área scrollable ───────────────────
        outer = tk.Frame(self.win, bg=BG)
        outer.pack(fill="both", expand=True)

        canvas = tk.Canvas(outer, bg=BG, highlightthickness=0)
        scrollbar = tk.Scrollbar(outer, orient="vertical", command=canvas.yview,
                                    bg=PANEL, troughcolor=BG)
        canvas.configure(yscrollcommand=scrollbar.set)

        scrollbar.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        interior = tk.Frame(canvas, bg=BG)
        canvas_window = canvas.create_window((0, 0), window=interior, anchor="nw")

        def on_configure(event):
            canvas.configure(scrollregion=canvas.bbox("all"))

        def on_canvas_resize(event):
            canvas.itemconfig(canvas_window, width=event.width)

        interior.bind("<Configure>", on_configure)
        canvas.bind("<Configure>", on_canvas_resize)

        # Habilitar scroll con rueda del mouse
        def on_mousewheel(event):
            canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")
        canvas.bind_all("<MouseWheel>", on_mousewheel)

        # Renderizar cada sección del manual
        for sec in MANUAL_SECCIONES:
            self._seccion(interior, sec)

        # ── Pie ───────────────────────────────
        pie = tk.Frame(self.win, bg=PANEL, pady=10)
        pie.pack(fill="x")
        tk.Label(pie, text="Análisis Numérico  •  Método de Punto Fijo  •  Segundo Parcial - 192349",
                    font=("Courier New", 8), fg=SUBTEXT, bg=PANEL).pack()

    def _seccion(self, parent, sec):
        """Renderiza un bloque de sección con título, separador y contenido."""
        bloque = tk.Frame(parent, bg=PANEL, padx=20, pady=14)
        bloque.pack(fill="x", padx=14, pady=(10, 0))

        titulo_frame = tk.Frame(bloque, bg=PANEL)
        titulo_frame.pack(fill="x", pady=(0, 8))

        tk.Label(titulo_frame, text=sec["icono"] + "  " + sec["titulo"],
                    font=("Courier New", 10, "bold"),
                    fg=sec["color"], bg=PANEL).pack(side="left")

        tk.Frame(bloque, height=1, bg=BORDER).pack(fill="x", pady=(0, 10))

        tk.Label(bloque, text=sec["contenido"],
                    font=("Courier New", 9),
                    fg=TEXT, bg=PANEL,
                    justify="left", wraplength=560,
                    anchor="w").pack(fill="x")


# ──────────────────────────────────────────────
#  APLICACIÓN PRINCIPAL
# ──────────────────────────────────────────────
class PuntoFijoApp:
    """
    Interfaz gráfica principal del método de punto fijo.
    El usuario ingresa f(x) y el programa busca automáticamente
    una g(x) convergente para resolver f(x) = 0.
    """
    def __init__(self, root):
        self.root = root
        self.root.title("Método de Punto Fijo — Análisis Numérico")
        self.root.configure(bg=BG)
        self.root.geometry("1150x760")
        self.root.resizable(True, True)
        self._build_ui()

    def _build_ui(self):
        """Construye la interfaz: encabezado, panel izquierdo y panel derecho."""
        # ── Encabezado ────────────────────────────────────────────────────
        header = tk.Frame(self.root, bg=BG, pady=14)
        header.pack(fill="x")
        tk.Label(header, text="●  MÉTODO DE PUNTO FIJO",
                    font=FONT_TITLE, fg=ACCENT, bg=BG).pack(side="left", padx=24)
        tk.Label(header, text="Análisis Numérico",
                    font=FONT_LABEL, fg=SUBTEXT, bg=BG).pack(side="left")

        # Botón de ayuda en la esquina superior derecha
        tk.Button(header, text="?  AYUDA",
                    font=("Courier New", 9, "bold"),
                    bg=ENTRY_BG, fg=ACCENT, relief="flat", cursor="hand2",
                    activebackground=BORDER, activeforeground=ACCENT,
                    padx=12, pady=4,
                    command=self.abrir_manual).pack(side="right", padx=20)

        tk.Frame(self.root, height=2, bg=BORDER).pack(fill="x")

        # ── Layout principal: columna izquierda y derecha ─────────────────
        main = tk.Frame(self.root, bg=BG)
        main.pack(fill="both", expand=True)

        left = tk.Frame(main, bg=PANEL, width=430)
        left.pack(side="left", fill="both", padx=(12, 6), pady=12)
        left.pack_propagate(False)

        right = tk.Frame(main, bg=BG)
        right.pack(side="left", fill="both", expand=True, padx=(6, 12), pady=12)

        # Construir secciones de la UI
        self._build_inputs(left)
        self._build_table(left)
        self._build_chart(right)

    def abrir_manual(self):
        """Abre la ventana del manual de instrucciones."""
        VentanaManual(self.root)

    def _build_inputs(self, parent):
        """
        Panel de entrada: campos para f(x), x₀, tolerancia,
        visualización de g(x) encontrada y botones de acción.
        """
        frame = tk.Frame(parent, bg=PANEL, padx=16, pady=14)
        frame.pack(fill="x")

        tk.Label(frame, text="PARÁMETROS DE ENTRADA",
                    font=("Courier New", 9, "bold"), fg=ACCENT, bg=PANEL).pack(anchor="w", pady=(0, 10))

        # ── Campo f(x) ────────────────────────────────────────────────────
        self._label(frame, "Función  f(x)  [se buscará g(x) automáticamente]")
        self.entry_fx = self._entry(frame)
        self.entry_fx.insert(0, "x**2 - x - 2")  # Ejemplo por defecto

        tk.Label(frame,
                 text="Ingresa f(x) tal que f(x) = 0   Ej: x**2 - x - 2  |  cos(x) - x  |  exp(-x) - x",
                    font=("Courier New", 8), fg=SUBTEXT, bg=PANEL, justify="left").pack(anchor="w", pady=(2, 8))

        # ── Valor inicial x₀ ─────────────────────────────────────────────
        self._label(frame, "Valor inicial  x₀")
        self.entry_x0 = self._entry(frame)
        self.entry_x0.insert(0, "1.5")

        # ── Tolerancia ───────────────────────────────────────────────────
        self._label(frame, "Tolerancia  ε")
        self.entry_tol = self._entry(frame)
        self.entry_tol.insert(0, "0.0001")

        # ── Visualización de g(x) encontrada (solo lectura) ──────────────
        self._label(frame, "g(x) encontrada  [se muestra después de calcular]")
        self.entry_gx_found = tk.Entry(
            frame, font=FONT_MONO, bg="#1a2a22", fg=SUCCESS,
            insertbackground=ACCENT, relief="flat",
            highlightthickness=1, highlightbackground=BORDER,
            highlightcolor=ACCENT, state="readonly"
        )
        self.entry_gx_found.pack(fill="x", ipady=5)

        # Etiqueta que indica la estrategia usada
        self.lbl_estrategia = tk.Label(
            frame, text="", font=("Courier New", 8), fg=ACCENT, bg=PANEL, justify="left"
        )
        self.lbl_estrategia.pack(anchor="w", pady=(2, 8))

        # ── Botones Calcular / Limpiar ────────────────────────────────────
        btn_frame = tk.Frame(frame, bg=PANEL, pady=10)
        btn_frame.pack(fill="x")

        tk.Button(btn_frame, text="▶  BUSCAR g(x) Y CALCULAR",
                    font=("Courier New", 10, "bold"),
                    bg=ACCENT, fg=BG, relief="flat", cursor="hand2",
                    activebackground="#00b894",
                    command=self.run).pack(side="left", fill="x", expand=True, ipady=6, padx=(0, 6))

        tk.Button(btn_frame, text="✕  LIMPIAR",
                    font=("Courier New", 10),
                    bg=ENTRY_BG, fg=SUBTEXT, relief="flat", cursor="hand2",
                    activebackground=BORDER,
                    command=self.clear).pack(side="left", ipady=6, ipadx=10)

        # ── Bloque de resultado final ─────────────────────────────────────
        res = tk.Frame(frame, bg=ENTRY_BG, padx=12, pady=10)
        res.pack(fill="x", pady=(4, 0))
        tk.Label(res, text="RESULTADO", font=("Courier New", 8, "bold"),
                    fg=SUBTEXT, bg=ENTRY_BG).pack(anchor="w")
        self.lbl_result = tk.Label(res, text="—", font=FONT_RESULT, fg=ACCENT, bg=ENTRY_BG)
        self.lbl_result.pack(anchor="w")
        self.lbl_detail = tk.Label(res, text="", font=("Courier New", 9), fg=SUBTEXT, bg=ENTRY_BG)
        self.lbl_detail.pack(anchor="w")

    def _build_table(self, parent):
        """Tabla de iteraciones con columnas n, xₙ, g(xₙ), Error."""
        frame = tk.Frame(parent, bg=PANEL, padx=16, pady=10)
        frame.pack(fill="both", expand=True, pady=(6, 0))

        tk.Label(frame, text="TABLA DE ITERACIONES",
                    font=("Courier New", 9, "bold"), fg=ACCENT, bg=PANEL).pack(anchor="w", pady=(0, 6))

        # Estilo oscuro para el Treeview
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("Custom.Treeview",
                        background=ENTRY_BG, foreground=TEXT,
                        fieldbackground=ENTRY_BG, rowheight=22,
                        font=("Courier New", 9), borderwidth=0)
        style.configure("Custom.Treeview.Heading",
                        background=BORDER, foreground=ACCENT,
                        font=("Courier New", 9, "bold"), relief="flat")
        style.map("Custom.Treeview", background=[("selected", ACCENT)])

        cols = ("n", "xn", "g(xn)", "Error")
        container = tk.Frame(frame, bg=PANEL)
        container.pack(fill="both", expand=True)

        self.tree = ttk.Treeview(container, columns=cols, show="headings",
                                    style="Custom.Treeview", selectmode="none")

        headers = ["n", "xₙ", "g(xₙ)", "Error (%)"]
        widths   = [38, 110, 110, 140]
        for col, hdr, w in zip(cols, headers, widths):
            self.tree.heading(col, text=hdr)
            self.tree.column(col, width=w, anchor="center")

        # Scrollbar vertical para la tabla
        scroll = tk.Scrollbar(container, orient="vertical",
                                command=self.tree.yview, bg=PANEL, troughcolor=PANEL)
        self.tree.configure(yscrollcommand=scroll.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scroll.pack(side="right", fill="y")

        # Colores de filas: alterna odd/even; última fila en verde
        self.tree.tag_configure("odd",  background="#1e2130", foreground=TEXT)
        self.tree.tag_configure("even", background=ENTRY_BG,  foreground=TEXT)
        self.tree.tag_configure("last", background="#1a3a2a",  foreground=SUCCESS)

    def _build_chart(self, parent):
        """Panel de gráficas: convergencia de xₙ y decrecimiento del error."""
        tk.Label(parent, text="GRÁFICA DE CONVERGENCIA",
                    font=("Courier New", 9, "bold"), fg=ACCENT, bg=BG).pack(anchor="w", pady=(6, 4))

        # Dos subplots apilados verticalmente
        self.fig, (self.ax1, self.ax2) = plt.subplots(2, 1, figsize=(6, 5.5),
                                                        facecolor=BG)
        self.fig.subplots_adjust(hspace=0.48, left=0.14, right=0.97, top=0.93, bottom=0.09)

        for ax in (self.ax1, self.ax2):
            ax.set_facecolor(PANEL)
            ax.tick_params(colors=SUBTEXT, labelsize=8)
            for spine in ax.spines.values():
                spine.set_color(BORDER)

        self.ax1.set_title("Valor xₙ por iteración", color=TEXT, fontsize=9, pad=6)
        self.ax2.set_title("Error (%) por iteración", color=TEXT, fontsize=9, pad=6)

        self.canvas = FigureCanvasTkAgg(self.fig, master=parent)
        self.canvas.get_tk_widget().pack(fill="both", expand=True)

    # ── Helpers de construcción de UI ─────────────────────────────────────

    def _label(self, parent, text):
        """Etiqueta de campo con estilo uniforme."""
        tk.Label(parent, text=text, font=FONT_LABEL,
                    fg=SUBTEXT, bg=PANEL).pack(anchor="w", pady=(6, 2))

    def _entry(self, parent):
        """Campo de entrada con estilo uniforme."""
        e = tk.Entry(parent, font=FONT_MONO, bg=ENTRY_BG, fg=TEXT,
                        insertbackground=ACCENT, relief="flat",
                        highlightthickness=1, highlightbackground=BORDER,
                        highlightcolor=ACCENT)
        e.pack(fill="x", ipady=5)
        return e

    # ── Lógica principal de cálculo ───────────────────────────────────────

    def run(self):
        """
        Callback del botón CALCULAR.
        1. Lee y valida los inputs.
        2. Llama a buscar_gx_convergente() para encontrar g(x).
        3. Muestra g(x) encontrada en la interfaz.
        4. Llena la tabla y actualiza las gráficas.
        """
        f_expr = self.entry_fx.get().strip()
        x0_str = self.entry_x0.get().strip()
        tol_str = self.entry_tol.get().strip()

        # ── Validaciones ──────────────────────────────────────────────────
        if not f_expr:
            messagebox.showerror("Error", "Ingresa la función f(x).")
            return
        try:
            x0 = float(x0_str)
        except ValueError:
            messagebox.showerror("Error", "x₀ debe ser un número válido.")
            return
        try:
            tol = float(tol_str)
            if tol <= 0:
                raise ValueError
        except ValueError:
            messagebox.showerror("Error", "La tolerancia debe ser un número positivo.")
            return

        # ── Búsqueda automática de g(x) ───────────────────────────────────
        resultado = buscar_gx_convergente(f_expr, x0, tol)

        if resultado is None:
            # Diagnóstico adicional: verificar si x0 está cerca de una asíntota
            f_prepro = preprocesar(f_expr)
            advertencia_asintota = cerca_de_asintota(f_prepro, x0)

            if advertencia_asintota:
                detalle = (
                    "⚠ x₀ parece estar cerca de una discontinuidad o asíntota de f(x).\n\n"
                    "Por ejemplo, tan(x) tiene asíntotas en x = π/2, 3π/2, ...\n"
                    "Intenta un x₀ alejado de esos puntos.\n\n"
                    "Sugerencias:\n"
                    "• Para tan(x) − x, prueba x₀ cercano a 0 (ej: 0.1)\n"
                    "• Verifica que f(x₀) esté bien definida y sea finita."
                )
            else:
                detalle = (
                    "No se encontró una g(x) que converja con el x₀ dado.\n\n"
                    "Sugerencias:\n"
                    "• Cambia x₀ a un valor más cercano a la raíz.\n"
                    "• Verifica que f(x) esté bien escrita.\n"
                    "• Asegúrate de que f(x) tenga una raíz cerca de x₀."
                )

            messagebox.showerror("Sin convergencia", detalle)
            return

        desc_gx, g_expr_usada, iters = resultado

        # ── Mostrar g(x) seleccionada en la interfaz ──────────────────────
        self.entry_gx_found.config(state="normal")
        self.entry_gx_found.delete(0, "end")
        self.entry_gx_found.insert(0, g_expr_usada)
        self.entry_gx_found.config(state="readonly")

        self.lbl_estrategia.config(
            text=f"Estrategia: {desc_gx}"
        )

        # ── Llenar tabla y gráficas ───────────────────────────────────────
        self._fill_table(iters)
        self._update_chart(iters)
        self._show_result(iters, tol)

    def _fill_table(self, iters):
        """Llena la tabla de iteraciones con los datos calculados."""
        # Limpiar filas anteriores
        for row in self.tree.get_children():
            self.tree.delete(row)

        for i, it in enumerate(iters):
            # La última fila se colorea en verde para indicar convergencia
            tag = "last" if i == len(iters) - 1 else ("odd" if i % 2 == 0 else "even")
            self.tree.insert("", "end", values=(
                it["n"],
                f"{it['xn']:.4f}",
                f"{it['gxn']:.4f}",
                f"{it['error'] * 100:.4f}%"
            ), tags=(tag,))

    def _update_chart(self, iters):
        """Actualiza las gráficas de convergencia de xₙ y error."""
        ns     = [it["n"]           for it in iters]
        xs     = [it["xn"]          for it in iters]
        errors = [it["error"] * 100  for it in iters]
        raiz   = iters[-1]["gxn"]  # Raíz aproximada = último g(xₙ)

        # ── Gráfica superior: evolución de xₙ ────────────────────────────
        self.ax1.clear()
        self.ax1.set_facecolor(PANEL)
        self.ax1.plot(ns, xs, color=ACCENT, linewidth=1.8, marker="o",
                        markersize=4, markerfacecolor=ACCENT2)
        self.ax1.axhline(raiz, color=SUCCESS, linewidth=1, linestyle="--",
                            alpha=0.7, label=f"x* ≈ {raiz:.4f}")
        self.ax1.set_title("Valor xₙ por iteración", color=TEXT, fontsize=9, pad=6)
        self.ax1.set_xlabel("Iteración", color=SUBTEXT, fontsize=8)
        self.ax1.set_ylabel("xₙ", color=SUBTEXT, fontsize=8)
        self.ax1.tick_params(colors=SUBTEXT, labelsize=8)
        self.ax1.legend(fontsize=7, facecolor=ENTRY_BG, edgecolor=BORDER, labelcolor=TEXT)
        for spine in self.ax1.spines.values():
            spine.set_color(BORDER)

        # ── Gráfica inferior: error por iteración ─────────────────────────
        self.ax2.clear()
        self.ax2.set_facecolor(PANEL)
        self.ax2.plot(ns, errors, color=ACCENT2, linewidth=1.8, marker="s",
                        markersize=4, markerfacecolor=ACCENT)
        self.ax2.set_title("Error (%) por iteración", color=TEXT, fontsize=9, pad=6)
        self.ax2.set_xlabel("Iteración", color=SUBTEXT, fontsize=8)
        self.ax2.set_ylabel("Error (%)", color=SUBTEXT, fontsize=8)
        self.ax2.tick_params(colors=SUBTEXT, labelsize=8)
        for spine in self.ax2.spines.values():
            spine.set_color(BORDER)

        self.canvas.draw()

    def _show_result(self, iters, tol):
        """Actualiza la etiqueta de resultado con la raíz encontrada."""
        last = iters[-1]
        convergio = last["error"] < tol
        self.lbl_result.config(
            text=f"x* ≈ {last['gxn']:.4f}",
            fg=SUCCESS if convergio else WARNING
        )
        estado = "Convergió ✓" if convergio else "No convergió — máx. iteraciones alcanzado"
        self.lbl_detail.config(
            text=f"{estado}  |  {last['n']} iter  |  ε = {last['error'] * 100:.4f}%"
        )

    def clear(self):
        """Restablece todos los campos e inputs a sus valores por defecto."""
        # Restaurar campos de texto
        self.entry_fx.delete(0, "end")
        self.entry_x0.delete(0, "end")
        self.entry_tol.delete(0, "end")
        self.entry_fx.insert(0, "x**2 - x - 2")
        self.entry_x0.insert(0, "1.5")
        self.entry_tol.insert(0, "0.0001")

        # Limpiar campo g(x) encontrada
        self.entry_gx_found.config(state="normal")
        self.entry_gx_found.delete(0, "end")
        self.entry_gx_found.config(state="readonly")
        self.lbl_estrategia.config(text="")

        # Limpiar tabla
        for row in self.tree.get_children():
            self.tree.delete(row)

        # Limpiar gráficas y restaurar títulos
        for ax in (self.ax1, self.ax2):
            ax.clear()
            ax.set_facecolor(PANEL)
            for spine in ax.spines.values():
                spine.set_color(BORDER)
        self.ax1.set_title("Valor xₙ por iteración", color=TEXT, fontsize=9, pad=6)
        self.ax2.set_title("Error (%) por iteración", color=TEXT, fontsize=9, pad=6)
        self.canvas.draw()

        # Restaurar etiquetas de resultado
        self.lbl_result.config(text="—", fg=ACCENT)
        self.lbl_detail.config(text="")


# ──────────────────────────────────────────────
#  PUNTO DE ENTRADA
# ──────────────────────────────────────────────
if __name__ == "__main__":
    root = tk.Tk()
    PuntoFijoApp(root)
    root.mainloop()
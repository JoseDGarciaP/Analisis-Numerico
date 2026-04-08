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
# ──────────────────────────────────────────────
def preprocesar(expr):
    expr = expr.strip()
    expr = re.sub(r'\bsen\b', 'sin', expr)
    expr = re.sub(r'\btg\b',  'tan', expr)
    expr = re.sub(r'\bln\b',  'log', expr)
    expr = re.sub(r'\be\s*\^\s*\(', 'exp(', expr)
    expr = re.sub(r'\be\s*\^\s*(-?[a-zA-Z0-9_.]+)', r'exp(\1)', expr)
    expr = re.sub(r'\^', '**', expr)
    return expr


# ──────────────────────────────────────────────
#  MÉTODO DE PUNTO FIJO
# ──────────────────────────────────────────────
def punto_fijo(g_expr, x0, tol, max_iter=100):
    g_expr = preprocesar(g_expr)

    allowed = {k: getattr(math, k) for k in dir(math) if not k.startswith("_")}
    allowed["abs"] = abs

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

        gx = round(gx, 4)
        x  = round(x, 4)

        error = abs((gx - x) / gx) if gx != 0 else abs(gx - x)
        iteraciones.append({"n": i, "xn": x, "gxn": gx, "error": error})

        if error <= tol:
            break
        x = gx

    return iteraciones


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
        "titulo": "Cómo usar la aplicación",
        "icono": "◉",
        "color": ACCENT,
        "contenido": (
            "1.  Ingresa g(x) en el campo 'Función g(x)'.\n"
            "        Ejemplo:  (x**2 + 2) / 3\n\n"
            "2.  Ingresa el valor inicial x₀ (punto de partida de la iteración).\n"
            "        Ejemplo:  1.0\n\n"
            "3.  Ingresa la tolerancia ε (criterio de parada por error porcentual).\n"
            "        Ejemplo:  0.0001  →  se detiene cuando el error < 0.01%\n\n"
            "4.  Presiona  ▶ CALCULAR  para ejecutar el método.\n\n"
            "5.  Presiona  ✕ LIMPIAR  para restablecer todos los campos."
        )
    },
    {
        "titulo": "Cómo escribir la función g(x)",
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
            "Ejemplos válidos:\n"
            "  (x**2 + 2) / 3\n"
            "  exp(-x)\n"
            "  e^(-x) - x            ← se convierte automáticamente\n"
            "  cos(x) / 2 + 1\n"
            "  sqrt(2*x + 3)\n"
            "  (log(x) + 5) / 4"
        )
    },
    {
        "titulo": "Interpretación de resultados",
        "icono": "◎",
        "color": SUCCESS,
        "contenido": (
            "TABLA DE ITERACIONES\n"
            "  n          →  Número de iteración\n"
            "  xₙ         →  Valor de x en la iteración actual\n"
            "  g(xₙ)      →  Valor calculado g(xₙ) = siguiente x\n"
            "  Error (%)  →  Error porcentual relativo: |g(xₙ)−xₙ| / |g(xₙ)| × 100\n"
            "  Fila verde →  Última iteración (donde se cumplió la tolerancia)\n\n"
            "RESULTADO FINAL\n"
            "  Verde  →  El método convergió: se encontró una raíz aproximada x*\n"
            "  Naranja →  No convergió en el máximo de iteraciones (100)\n\n"
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
            "Si el método no converge (valores que crecen o oscilan), intenta:\n\n"
            "  • Reescribir f(x) = 0 de otra forma para obtener una g(x) diferente.\n"
            "  • Cambiar el valor inicial x₀, más cercano a la raíz esperada.\n"
            "  • Verificar que la función esté bien escrita (paréntesis, operadores).\n\n"
            "Ejemplo: para x² − x − 2 = 0  se puede usar:\n"
            "  g(x) = x² − 2    ← puede no converger\n"
            "  g(x) = sqrt(x+2) ← converge para x₀ > 0"
        )
    },
    {
        "titulo": "Errores comunes",
        "icono": "⚠",
        "color": ACCENT2,
        "contenido": (
            "• 'unsupported operand type ^ float'\n"
            "  →  Usa ** en lugar de ^ para potencias en Python,\n"
            "     o simplemente escribe e^x (se convierte automáticamente).\n\n"
            "• 'math domain error'\n"
            "  →  La función no está definida para ese x (ej: sqrt de negativo,\n"
            "     log de cero o negativo). Cambia x₀.\n\n"
            "• La tabla muestra solo 1 fila sin convergir\n"
            "  →  La función diverge desde x₀. Prueba otro valor inicial.\n\n"
            "• Resultado muy diferente al esperado\n"
            "  →  Revisa los paréntesis. Ej:  x+2/3  ≠  (x+2)/3"
        )
    },
]


class VentanaManual:
    def __init__(self, parent):
        self.win = tk.Toplevel(parent)
        self.win.title("Manual de Instrucciones")
        self.win.configure(bg=BG)
        self.win.geometry("640x680")
        self.win.resizable(True, True)
        # No bloquea la ventana principal
        self.win.transient(parent)
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
        outer.pack(fill="both", expand=True, padx=0, pady=0)

        canvas = tk.Canvas(outer, bg=BG, highlightthickness=0)
        scrollbar = tk.Scrollbar(outer, orient="vertical", command=canvas.yview,
                                    bg=PANEL, troughcolor=BG)
        canvas.configure(yscrollcommand=scrollbar.set)

        scrollbar.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        # Frame interior dentro del canvas
        interior = tk.Frame(canvas, bg=BG)
        canvas_window = canvas.create_window((0, 0), window=interior, anchor="nw")

        def on_configure(event):
            canvas.configure(scrollregion=canvas.bbox("all"))

        def on_canvas_resize(event):
            canvas.itemconfig(canvas_window, width=event.width)

        interior.bind("<Configure>", on_configure)
        canvas.bind("<Configure>", on_canvas_resize)

        # Scroll con rueda del mouse
        def on_mousewheel(event):
            canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")
        canvas.bind_all("<MouseWheel>", on_mousewheel)

        # ── Secciones ─────────────────────────
        for sec in MANUAL_SECCIONES:
            self._seccion(interior, sec)

        # ── Pie ───────────────────────────────
        pie = tk.Frame(self.win, bg=PANEL, pady=10)
        pie.pack(fill="x")
        tk.Label(pie, text="Análisis Numérico  •  Método de Punto Fijo  •  Segundo Parcial - 192349",
                    font=("Courier New", 8), fg=SUBTEXT, bg=PANEL).pack()

    def _seccion(self, parent, sec):
        bloque = tk.Frame(parent, bg=PANEL, padx=20, pady=14)
        bloque.pack(fill="x", padx=14, pady=(10, 0))

        # Título de sección
        titulo_frame = tk.Frame(bloque, bg=PANEL)
        titulo_frame.pack(fill="x", pady=(0, 8))

        tk.Label(titulo_frame, text=sec["icono"] + "  " + sec["titulo"],
                    font=("Courier New", 10, "bold"),
                    fg=sec["color"], bg=PANEL).pack(side="left")

        # Línea separadora bajo el título
        tk.Frame(bloque, height=1, bg=BORDER).pack(fill="x", pady=(0, 10))

        # Contenido
        tk.Label(bloque, text=sec["contenido"],
                    font=("Courier New", 9),
                    fg=TEXT, bg=PANEL,
                    justify="left", wraplength=560,
                    anchor="w").pack(fill="x")


# ──────────────────────────────────────────────
#  APLICACIÓN PRINCIPAL
# ──────────────────────────────────────────────
class PuntoFijoApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Método de Punto Fijo — Análisis Numérico")
        self.root.configure(bg=BG)
        self.root.geometry("1150x720")
        self.root.resizable(True, True)
        self._build_ui()

    def _build_ui(self):
        header = tk.Frame(self.root, bg=BG, pady=14)
        header.pack(fill="x")
        tk.Label(header, text="●  MÉTODO DE PUNTO FIJO",
                    font=FONT_TITLE, fg=ACCENT, bg=BG).pack(side="left", padx=24)
        tk.Label(header, text="Análisis Numérico",
                    font=FONT_LABEL, fg=SUBTEXT, bg=BG).pack(side="left")

        # Botón de ayuda en el encabezado
        tk.Button(header, text="?  AYUDA",
                    font=("Courier New", 9, "bold"),
                    bg=ENTRY_BG, fg=ACCENT, relief="flat", cursor="hand2",
                    activebackground=BORDER, activeforeground=ACCENT,
                    padx=12, pady=4,
                    command=self.abrir_manual).pack(side="right", padx=20)

        tk.Frame(self.root, height=2, bg=BORDER).pack(fill="x")

        main = tk.Frame(self.root, bg=BG)
        main.pack(fill="both", expand=True)

        left = tk.Frame(main, bg=PANEL, width=430)
        left.pack(side="left", fill="both", padx=(12, 6), pady=12)
        left.pack_propagate(False)

        right = tk.Frame(main, bg=BG)
        right.pack(side="left", fill="both", expand=True, padx=(6, 12), pady=12)

        self._build_inputs(left)
        self._build_table(left)
        self._build_chart(right)

    def abrir_manual(self):
        VentanaManual(self.root)

    def _build_inputs(self, parent):
        frame = tk.Frame(parent, bg=PANEL, padx=16, pady=14)
        frame.pack(fill="x")

        tk.Label(frame, text="PARÁMETROS DE ENTRADA",
                    font=("Courier New", 9, "bold"), fg=ACCENT, bg=PANEL).pack(anchor="w", pady=(0, 10))

        self._label(frame, "Función  g(x)")
        self.entry_gx = self._entry(frame)
        self.entry_gx.insert(0, "(x**2 + 2) / 3")

        tk.Label(frame,
                 text="sin/cos/tan/exp/sqrt/log  |  ^ o ** para potencia  |  e^x se convierte solo\n"
                        "También aceptado: sen, tg, ln   Ej: e^(-x)-x  ó  exp(-x)-x",
                    font=("Courier New", 8), fg=SUBTEXT, bg=PANEL, justify="left").pack(anchor="w", pady=(2, 8))

        self._label(frame, "Valor inicial  x₀")
        self.entry_x0 = self._entry(frame)
        self.entry_x0.insert(0, "1.0")

        self._label(frame, "Tolerancia  ε")
        self.entry_tol = self._entry(frame)
        self.entry_tol.insert(0, "0.0001")

        btn_frame = tk.Frame(frame, bg=PANEL, pady=10)
        btn_frame.pack(fill="x")

        tk.Button(btn_frame, text="▶  CALCULAR",
                    font=("Courier New", 10, "bold"),
                    bg=ACCENT, fg=BG, relief="flat", cursor="hand2",
                    activebackground="#00b894",
                    command=self.run).pack(side="left", fill="x", expand=True, ipady=6, padx=(0, 6))

        tk.Button(btn_frame, text="✕  LIMPIAR",
                    font=("Courier New", 10),
                    bg=ENTRY_BG, fg=SUBTEXT, relief="flat", cursor="hand2",
                    activebackground=BORDER,
                    command=self.clear).pack(side="left", ipady=6, ipadx=10)

        res = tk.Frame(frame, bg=ENTRY_BG, padx=12, pady=10)
        res.pack(fill="x", pady=(4, 0))
        tk.Label(res, text="RESULTADO", font=("Courier New", 8, "bold"),
                    fg=SUBTEXT, bg=ENTRY_BG).pack(anchor="w")
        self.lbl_result = tk.Label(res, text="—", font=FONT_RESULT, fg=ACCENT, bg=ENTRY_BG)
        self.lbl_result.pack(anchor="w")
        self.lbl_detail = tk.Label(res, text="", font=("Courier New", 9), fg=SUBTEXT, bg=ENTRY_BG)
        self.lbl_detail.pack(anchor="w")

    def _build_table(self, parent):
        frame = tk.Frame(parent, bg=PANEL, padx=16, pady=10)
        frame.pack(fill="both", expand=True, pady=(6, 0))

        tk.Label(frame, text="TABLA DE ITERACIONES",
                    font=("Courier New", 9, "bold"), fg=ACCENT, bg=PANEL).pack(anchor="w", pady=(0, 6))

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

        scroll = tk.Scrollbar(container, orient="vertical",
                                command=self.tree.yview, bg=PANEL, troughcolor=PANEL)
        self.tree.configure(yscrollcommand=scroll.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scroll.pack(side="right", fill="y")

        self.tree.tag_configure("odd",  background="#1e2130", foreground=TEXT)
        self.tree.tag_configure("even", background=ENTRY_BG,  foreground=TEXT)
        self.tree.tag_configure("last", background="#1a3a2a",  foreground=SUCCESS)

    def _build_chart(self, parent):
        tk.Label(parent, text="GRÁFICA DE CONVERGENCIA",
                    font=("Courier New", 9, "bold"), fg=ACCENT, bg=BG).pack(anchor="w", pady=(6, 4))

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

    def _label(self, parent, text):
        tk.Label(parent, text=text, font=FONT_LABEL,
                    fg=SUBTEXT, bg=PANEL).pack(anchor="w", pady=(6, 2))

    def _entry(self, parent):
        e = tk.Entry(parent, font=FONT_MONO, bg=ENTRY_BG, fg=TEXT,
                        insertbackground=ACCENT, relief="flat",
                        highlightthickness=1, highlightbackground=BORDER,
                        highlightcolor=ACCENT)
        e.pack(fill="x", ipady=5)
        return e

    def run(self):
        g_expr = self.entry_gx.get().strip()
        x0_str = self.entry_x0.get().strip()
        tol_str = self.entry_tol.get().strip()

        if not g_expr:
            messagebox.showerror("Error", "Ingresa la función g(x).")
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

        try:
            iters = punto_fijo(g_expr, x0, tol)
        except Exception as e:
            messagebox.showerror("Error en g(x)", str(e))
            return

        self._fill_table(iters)
        self._update_chart(iters)
        self._show_result(iters, tol)

    def _fill_table(self, iters):
        for row in self.tree.get_children():
            self.tree.delete(row)
        for i, it in enumerate(iters):
            tag = "last" if i == len(iters) - 1 else ("odd" if i % 2 == 0 else "even")
            self.tree.insert("", "end", values=(
                it["n"],
                f"{it['xn']:.4f}",
                f"{it['gxn']:.4f}",
                f"{it['error'] * 100:.4f}%"
            ), tags=(tag,))

    def _update_chart(self, iters):
        ns     = [it["n"]          for it in iters]
        xs     = [it["xn"]         for it in iters]
        errors = [it["error"] * 100 for it in iters]
        raiz   = iters[-1]["gxn"]

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
        self.entry_gx.delete(0, "end")
        self.entry_x0.delete(0, "end")
        self.entry_tol.delete(0, "end")
        self.entry_gx.insert(0, "(x**2 + 2) / 3")
        self.entry_x0.insert(0, "1.0")
        self.entry_tol.insert(0, "0.0001")

        for row in self.tree.get_children():
            self.tree.delete(row)

        for ax in (self.ax1, self.ax2):
            ax.clear()
            ax.set_facecolor(PANEL)
            for spine in ax.spines.values():
                spine.set_color(BORDER)
        self.ax1.set_title("Valor xₙ por iteración", color=TEXT, fontsize=9, pad=6)
        self.ax2.set_title("Error (%) por iteración", color=TEXT, fontsize=9, pad=6)
        self.canvas.draw()

        self.lbl_result.config(text="—", fg=ACCENT)
        self.lbl_detail.config(text="")


# ──────────────────────────────────────────────
if __name__ == "__main__":
    root = tk.Tk()
    PuntoFijoApp(root)
    root.mainloop()
# Rampa en U paramétrica — GDL v0.6 (completa, sin probar en Archicad)

Cambios respecto a la v0.5 (revisión 2):
- **Nivel 0.00 = cara superior terminada de la rampa** (revestimiento si existe, si no el núcleo); el núcleo y el revestimiento se extienden por debajo.
- **Los extremos son cortes verticales** (perpendiculares al piso) en núcleo, revestimiento y parapetos; ya no hay cuña paralela al piso.
- `ZZYZX` ahora es la altura hasta la cara superior terminada.
- Texto: base hacia la derecha en ambos tramos y línea de paso entre las dos líneas de texto.
- `macizo` ya no se usa (puedes borrarlo).
- Marca de corte y línea de paso ahora son casillas.
- Texto fijo en cada tramo: pendiente y longitud inclinada (suma de los lados inclinados, sin descansos), con fuente, tamaño, pluma y orientación configurables.
- Parámetros bloqueados cuando no aplican (parapeto, texto, rotura simbólica).
- Un solo juego de parapetos a ambos lados (como ya estaba).

## Parámetros nuevos respecto a la v0.5
| Nombre | Tipo | Valor | Descripción |
|---|---|---|---|
| mostrarRotura | Booleano | 1 | Mostrar la marca de corte |
| mostrarRecorrido | Booleano | 1 | Mostrar la línea de paso (círculo, trazo y flecha) |
| fuenteTexto | Texto | Arial | Nombre de la fuente |
| tamTexto | Número real | 2 | Tamaño del texto en mm de papel |
| penTexto | Pluma | 1 | Pluma del texto |
| orientTexto | Booleano | 1 | Activado = alineado con el símbolo (base a la derecha) · desactivado = Legible (se voltea si el símbolo girado lo dejaría al revés) |

Ajustes en parámetros que ya tienes:
- **`revestir` debe ser Booleano.** Si lo creaste como Entero, cámbiale el tipo en la lista de parámetros (si no te deja, bórralo y créalo de nuevo como Booleano, valor 0).
- Revisa que `parapeto`, `macizo`, `descIntermedio`, `mostrarPend`, `rotSimbolo` también sean Booleanos.
- `mostrarPend` ahora activa o desactiva todo el texto (pendiente y longitud). Renombra su etiqueta a "Mostrar texto".
- `fillSel` (Relleno vacío) sigue igual que en la v0.5.2.

Los cuatro scripts están completos abajo. También van como archivos de texto separados (uno por script) para copiar y pegar enteros.

## Script de parámetros
```gdl
VALUES{2} "vincAltura", 1, "Sin vinculación (altura manual)", 2, "Vinculada a planta superior"
VALUES "plantaSup" RANGE [1,)
VALUES "ZZYZX" RANGE (0,)
VALUES "anchoTramo" RANGE (0,)
VALUES "ojo" RANGE [0,)
VALUES "dDescanso" RANGE [anchoTramo,)
VALUES "pend" RANGE [1,40]
VALUES "esp" RANGE (0,)
VALUES "nDescInt" 1, 2, 3
VALUES "dInt" RANGE (0,)

IF vincAltura = 2 THEN
    LOCK "ZZYZX"
ELSE
    LOCK "plantaSup", "offsetSup"
ENDIF

IF revestir THEN
    VALUES "espRev" RANGE (0,)
ELSE
    LOCK "espRev", "matRev"
ENDIF

IF rotSimbolo = 0 THEN LOCK "ltRotura"
IF parapeto = 0 THEN LOCK "espPar", "altPar", "matPar"
IF mostrarPend = 0 THEN LOCK "fuenteTexto", "tamTexto", "penTexto", "orientTexto"
VALUES "tamTexto" RANGE (0,)
DIM fontNames[]
DIM fontCorta[]
nF = REQUEST ("FONTNAMES_LIST", "", fontNames)
kF = 0
FOR idxF = 1 TO VARDIM1 (fontNames)
    IF STRSTR (fontNames[idxF], "@") <> 1 THEN
        kF = kF + 1
        fontCorta[kF] = fontNames[idxF]
    ENDIF
NEXT idxF
IF kF > 0 THEN VALUES "fuenteTexto" fontCorta, CUSTOM

IF descIntermedio THEN
    nDI = MIN(MAX(nDescInt, 1), 3)
ELSE
    nDI = 0
ENDIF

lf_ = (ZZYZX / 2) / (pend / 100) + nDI * dInt
PARAMETERS largoTramo = lf_
PARAMETERS A = 2 * anchoTramo + ojo
PARAMETERS B = lf_ + dDescanso
LOCK "A", "B", "largoTramo"
```

## Master
```gdl
! --- Altura total: manual o vinculada a la planta superior
alturaT = ZZYZX
IF vincAltura = 2 THEN
    rq = REQUEST ("Home_story", "", idxHome, nomHome)
    rq = REQUEST ("Story_info", idxHome, nPl1, idx1, nom1, elev1, alt1)
    rq = REQUEST ("Story_info", idxHome + plantaSup, nPl2, idx2, nom2, elev2, alt2)
    hVinc = elev2 - elev1 - GLOB_ELEVATION + offsetSup
    IF hVinc > 0.1 THEN alturaT = hVinc
ENDIF

hMed = alturaT / 2
Ht   = alturaT
Lt   = hMed / (pend / 100)      ! recorrido inclinado en planta (sin descansos)
wA   = anchoTramo
Atot = 2 * wA + ojo

IF descIntermedio THEN
    nDI = MIN(MAX(nDescInt, 1), 3)
ELSE
    nDI = 0
ENDIF

hs = hMed / (nDI + 1)           ! desnivel de cada rampa parcial
Ls = Lt / (nDI + 1)             ! largo en planta de cada rampa parcial
LF = Lt + nDI * dInt            ! largo en planta de cada tramo
Btot = LF + dDescanso
Linc = SQR(Lt * Lt + hMed * hMed)     ! longitud inclinada de cada tramo, sin descansos

! Revestimiento y parapeto
eRev = 0
IF revestir AND espRev > 0.0005 THEN eRev = espRev
pe = 0
IF parapeto THEN pe = espPar
altPar = altPar + eRev          ! altura de parapeto medida desde la cara superior terminada

! Perfil de la CARA SUPERIOR TERMINADA del tramo 1: (uT, zT). Nivel 0.00 = esta cara en el arranque;
! el núcleo y el revestimiento se extienden por debajo.
DIM uT[8], zT[8]
uT[1] = 0
zT[1] = 0
np = 1
FOR i = 1 TO nDI + 1
    np = np + 1
    uT[np] = uT[np - 1] + Ls
    zT[np] = zT[np - 1] + hs
    IF i <= nDI THEN
        np = np + 1
        uT[np] = uT[np - 1] + dInt
        zT[np] = zT[np - 1]
    ENDIF
NEXT i

! Parámetros calculados / niveles del objeto
IF vincAltura = 2 THEN PARAMETERS ZZYZX = alturaT
PARAMETERS A = Atot, B = Btot, largoTramo = LF
PARAMETERS ac_bottomlevel = -(esp + eRev), ac_toplevel = alturaT + altPar
```

## 3D
```gdl
! --- Hotspots de edición (id únicos)
idH = 1
! Ancho de tramos: borde exterior izquierdo (base editable) y borde interior del tramo 1
HOTSPOT 0, 0, 0, idH, anchoTramo, 1 + 256 : idH = idH + 1
HOTSPOT -1, 0, 0, idH, anchoTramo, 3 : idH = idH + 1
HOTSPOT wA, 0, 0, idH, anchoTramo, 2 : idH = idH + 1
HOTSPOT 0, LF, hMed, idH, anchoTramo, 1 + 256 : idH = idH + 1
HOTSPOT -1, LF, hMed, idH, anchoTramo, 3 : idH = idH + 1
HOTSPOT wA, LF, hMed, idH, anchoTramo, 2 : idH = idH + 1
! Ancho del ojo: borde interior del tramo 2
HOTSPOT wA, 0, 0, idH, ojo, 1 + 128 : idH = idH + 1
HOTSPOT wA - 1, 0, 0, idH, ojo, 3 : idH = idH + 1
HOTSPOT wA + ojo, 0, 0, idH, ojo, 2 : idH = idH + 1
HOTSPOT wA, LF, hMed, idH, ojo, 1 + 128 : idH = idH + 1
HOTSPOT wA - 1, LF, hMed, idH, ojo, 3 : idH = idH + 1
HOTSPOT wA + ojo, LF, hMed, idH, ojo, 2 : idH = idH + 1
! Puntos de referencia
HOTSPOT Atot, 0, 0
HOTSPOT Atot, Btot, 0
HOTSPOT 0, Btot, 0
HOTSPOT 0, 0, Ht

! ===================== NÚCLEO =====================
! Nivel 0.00 = cara superior de la rampa (revestimiento si existe, si no el núcleo).
! El cuerpo se extiende por debajo y los extremos son cortes verticales.
SET BUILDING_MATERIAL matRampa

! --- Tramo 1 (X = 0 .. wA), sube hacia +Y
FOR i = 1 TO np
    PUT uT[i], zT[i] - eRev, 15
NEXT i
FOR i = np TO 1 STEP -1
    PUT uT[i], zT[i] - eRev - esp, 15
NEXT i
exW = wA
GOSUB "extruir"

! --- Tramo 2 (X = wA+ojo .. Atot), sube hacia -Y (perfil espejado y elevado hMed)
FOR j = 1 TO np
    PUT LF - uT[np + 1 - j], hMed + zT[np + 1 - j] - eRev, 15
NEXT j
FOR j = np TO 1 STEP -1
    PUT LF - uT[np + 1 - j], hMed + zT[np + 1 - j] - eRev - esp, 15
NEXT j
exW = wA
ADDX wA + ojo
GOSUB "extruir"
DEL 1

! --- Descanso de giro (núcleo)
ADD 0, LF, hMed - eRev - esp
BLOCK Atot, dDescanso, esp
DEL 1

! ===================== PARAPETOS (sobre el núcleo) =====================
IF parapeto THEN
    SET BUILDING_MATERIAL matPar
    exW = espPar

    GOSUB "parTramo1"
    GOSUB "extruir"
    GOSUB "parTramo1"
    ADDX wA - espPar
    GOSUB "extruir"
    DEL 1
    GOSUB "parTramo2"
    ADDX wA + ojo
    GOSUB "extruir"
    DEL 1
    GOSUB "parTramo2"
    ADDX Atot - espPar
    GOSUB "extruir"
    DEL 1

    ! Descanso: laterales y fondo
    ADD 0, LF, hMed - eRev
    BLOCK espPar, dDescanso, altPar
    DEL 1
    ADD Atot - espPar, LF, hMed - eRev
    BLOCK espPar, dDescanso, altPar
    DEL 1
    ADD espPar, Btot - espPar, hMed - eRev
    BLOCK Atot - 2 * espPar, espPar, altPar
    DEL 1

    ! Cierre del ojo: cubre también las esquinas de los parapetos interiores
    ADD wA - espPar, LF, hMed - eRev
    BLOCK ojo + 2 * espPar, espPar, altPar
    DEL 1
ENDIF

! ===================== REVESTIMIENTO (entre parapetos) =====================
IF eRev > 0.0005 THEN
    SET BUILDING_MATERIAL matRev
    wF = wA - 2 * pe

    IF wF > 0.001 THEN
        ! Tramo 1
        FOR i = 1 TO np
            PUT uT[i], zT[i], 15
        NEXT i
        FOR i = np TO 1 STEP -1
            PUT uT[i], zT[i] - eRev, 15
        NEXT i
        exW = wF
        ADDX pe
        GOSUB "extruir"
        DEL 1

        ! Tramo 2
        FOR j = 1 TO np
            PUT LF - uT[np + 1 - j], hMed + zT[np + 1 - j], 15
        NEXT j
        FOR j = np TO 1 STEP -1
            PUT LF - uT[np + 1 - j], hMed + zT[np + 1 - j] - eRev, 15
        NEXT j
        exW = wF
        ADDX wA + ojo + pe
        GOSUB "extruir"
        DEL 1
    ENDIF

    ! Descanso de giro
    IF parapeto THEN
        ADD pe, LF + pe, hMed - eRev
        BLOCK Atot - 2 * pe, dDescanso - 2 * pe, eRev
        DEL 1
        IF wF > 0.001 THEN
            ADD pe, LF, hMed - eRev
            BLOCK wF, pe, eRev
            DEL 1
            ADD wA + ojo + pe, LF, hMed - eRev
            BLOCK wF, pe, eRev
            DEL 1
        ENDIF
    ELSE
        ADD 0, LF, hMed - eRev
        BLOCK Atot, dDescanso, eRev
        DEL 1
    ENDIF
ENDIF

END

! ---------- Subrutinas 3D ----------
! Extruye el perfil guardado con PUT (x = recorrido en planta Y, y = cota Z) a lo largo de +X
"extruir":
    ROTZ 90
    ROTX 90
    PRISM_ NSP/3, exW, GET(NSP)
    DEL 2
RETURN

! Parapeto: desde la cara superior del núcleo hasta altPar por encima
"parTramo1":
    FOR i = 1 TO np
        PUT uT[i], zT[i] - eRev, 15
    NEXT i
    FOR i = np TO 1 STEP -1
        PUT uT[i], zT[i] - eRev + altPar, 15
    NEXT i
RETURN

"parTramo2":
    FOR j = 1 TO np
        PUT LF - uT[np + 1 - j], hMed + zT[np + 1 - j] - eRev, 15
    NEXT j
    FOR j = np TO 1 STEP -1
        PUT LF - uT[np + 1 - j], hMed + zT[np + 1 - j] - eRev + altPar, 15
    NEXT j
RETURN
```

## 2D
```gdl
DIM ra[6], rd[6]

! --- Hotspots de edición (mismos que en 3D)
idH = 1
HOTSPOT2 0, 0, idH, anchoTramo, 1 + 256 : idH = idH + 1
HOTSPOT2 -1, 0, idH, anchoTramo, 3 : idH = idH + 1
HOTSPOT2 wA, 0, idH, anchoTramo, 2 : idH = idH + 1
HOTSPOT2 0, LF, idH, anchoTramo, 1 + 256 : idH = idH + 1
HOTSPOT2 -1, LF, idH, anchoTramo, 3 : idH = idH + 1
HOTSPOT2 wA, LF, idH, anchoTramo, 2 : idH = idH + 1
HOTSPOT2 wA, 0, idH, ojo, 1 + 128 : idH = idH + 1
HOTSPOT2 wA - 1, 0, idH, ojo, 3 : idH = idH + 1
HOTSPOT2 wA + ojo, 0, idH, ojo, 2 : idH = idH + 1
HOTSPOT2 wA, LF, idH, ojo, 1 + 128 : idH = idH + 1
HOTSPOT2 wA - 1, LF, idH, ojo, 3 : idH = idH + 1
HOTSPOT2 wA + ojo, LF, idH, ojo, 2 : idH = idH + 1
HOTSPOT2 Atot, 0
HOTSPOT2 Atot, Btot
HOTSPOT2 0, Btot

PEN gs_cont_pen

! --- Plano de corte (altura local respecto al origen del objeto)
hc = GLOB_CUTPLANES_INFO[1]
kb = TAN(30)                    ! inclinación de la línea de rotura
dk = kb * wA / 2

zbT = 0
GOSUB "estadoTramo"
st1 = stT : sc1 = scT
GOSUB "bordes"
sIn1 = sInter : sOut1 = sExter

zbT = hMed
GOSUB "estadoTramo"
st2 = stT : sc2 = scT
GOSUB "bordes"
sIn2 = sInter : sOut2 = sExter

! --- Área de selección (relleno vacío, transparente): huella completa en U
FILL fillSel                     ! relleno vacío elegido en el parámetro fillSel
PUT 0, 0, 1
PUT wA, 0, 1
PUT wA, LF, 1
PUT wA + ojo, LF, 1
PUT wA + ojo, 0, 1
PUT Atot, 0, 1
PUT Atot, Btot, 1
PUT 0, Btot, 1
POLY2_B NSP / 3, 2 + 4, gs_fill_pen, 0, GET(NSP)

! --- Relleno solo bajo el plano de corte (transparente sobre el corte)
FILL gs_fill_type
IF st1 > 0 THEN
    PUT 0, 0, 1
    PUT wA, 0, 1
    PUT wA, sIn1, 1
    PUT 0, sOut1, 1
    POLY2_B NSP / 3, 2 + 4, gs_fill_pen, gs_back_pen, GET(NSP)
ENDIF
IF st2 > 0 THEN
    PUT wA + ojo, LF, 1
    PUT Atot, LF, 1
    PUT Atot, LF - sOut2, 1
    PUT wA + ojo, LF - sIn2, 1
    POLY2_B NSP / 3, 2 + 4, gs_fill_pen, gs_back_pen, GET(NSP)
ENDIF
IF hMed < hc THEN
    PUT 0, LF, 1
    PUT Atot, LF, 1
    PUT Atot, Btot, 1
    PUT 0, Btot, 1
    POLY2_B NSP / 3, 2 + 4, gs_fill_pen, gs_back_pen, GET(NSP)
ENDIF

PEN gs_cont_pen

! --- Bordes de los tramos (continuo bajo el corte, discontinuo sobre el corte)
fid = 1
lx = 0 : sT = sOut1 : GOSUB "ladoTramo"
lx = wA : sT = sIn1 : GOSUB "ladoTramo"
fid = 2
lx = Atot : sT = sOut2 : GOSUB "ladoTramo"
lx = wA + ojo : sT = sIn2 : GOSUB "ladoTramo"
IF parapeto THEN
    fid = 1
    lx = espPar : sT = sOut1 : GOSUB "ladoTramo"
    lx = wA - espPar : sT = sIn1 : GOSUB "ladoTramo"
    fid = 2
    lx = Atot - espPar : sT = sOut2 : GOSUB "ladoTramo"
    lx = wA + ojo + espPar : sT = sIn2 : GOSUB "ladoTramo"
ENDIF

! --- Líneas transversales de los tramos
tx1 = 0 : tx2 = wA
ty1 = 0 : ty2 = 0 : tz = 0 : GOSUB "trAuto"
ty1 = LF : ty2 = LF : tz = hMed : GOSUB "trAuto"
tx1 = wA + ojo : tx2 = Atot
ty1 = LF : ty2 = LF : tz = hMed : GOSUB "trAuto"
ty1 = 0 : ty2 = 0 : tz = Ht : GOSUB "trAuto"

! --- Límites de los descansos intermedios
FOR k = 1 TO nDI
    u1 = k * Ls + (k - 1) * dInt
    u2 = u1 + dInt
    tx1 = 0 : tx2 = wA
    tz = k * hs
    ty1 = u1 : ty2 = u1 : GOSUB "trAuto"
    ty1 = u2 : ty2 = u2 : GOSUB "trAuto"
    tx1 = wA + ojo : tx2 = Atot
    tz = hMed + k * hs
    ty1 = LF - u1 : ty2 = LF - u1 : GOSUB "trAuto"
    ty1 = LF - u2 : ty2 = LF - u2 : GOSUB "trAuto"
NEXT k

! --- Descanso de giro
tz = hMed
tx1 = 0 : ty1 = LF : tx2 = 0 : ty2 = Btot : GOSUB "trAuto"
tx1 = Atot : ty1 = LF : tx2 = Atot : ty2 = Btot : GOSUB "trAuto"
tx1 = 0 : ty1 = Btot : tx2 = Atot : ty2 = Btot : GOSUB "trAuto"
IF ojo > 0.001 THEN
    tx1 = wA : ty1 = LF : tx2 = wA + ojo : ty2 = LF : GOSUB "trAuto"
ENDIF
IF parapeto THEN
    tx1 = espPar : ty1 = LF : tx2 = espPar : ty2 = Btot - espPar : GOSUB "trAuto"
    tx1 = Atot - espPar : ty1 = LF : tx2 = Atot - espPar : ty2 = Btot - espPar : GOSUB "trAuto"
    tx1 = espPar : ty1 = Btot - espPar : tx2 = Atot - espPar : ty2 = Btot - espPar : GOSUB "trAuto"
    ! Cierre del ojo: rectángulo que une los dos parapetos interiores
    tx1 = wA - espPar : ty1 = LF + espPar : tx2 = wA + ojo + espPar : ty2 = LF + espPar : GOSUB "trAuto"
    tx1 = wA - espPar : ty1 = LF : tx2 = wA - espPar : ty2 = LF + espPar : GOSUB "trAuto"
    tx1 = wA + ojo + espPar : ty1 = LF : tx2 = wA + ojo + espPar : ty2 = LF + espPar : GOSUB "trAuto"
ENDIF

! --- Líneas de rotura donde el plano de corte atraviesa un tramo
IF st1 = 2 AND mostrarRotura THEN
    fidB = 1 : scB = sc1 : xcB = wA / 2 : sgB = 1
    GOSUB "rotura"
ENDIF
IF st2 = 2 AND mostrarRotura THEN
    fidB = 2 : scB = sc2 : xcB = Atot - wA / 2 : sgB = -1
    GOSUB "rotura"
ENDIF

! --- Línea de recorrido: círculo de inicio y flecha de llegada
xc1 = wA / 2
xc2 = Atot - wA / 2
IF mostrarRecorrido THEN
    PEN gs_cont_pen
    yP = LF + dDescanso / 2
    LINE_TYPE 1
    CIRCLE2 xc1, 0, 0.1
    LINE2 xc1, 0, xc1, yP
    LINE2 xc1, yP, xc2, yP
    LINE2 xc2, yP, xc2, 0
    ah = MIN(0.45 * wA, 0.5)
    LINE2 xc2, 0, xc2 - ah, ah
    LINE2 xc2, 0, xc2 + ah, ah
ENDIF

! --- Texto de cada tramo: pendiente y longitud inclinada (sin descansos), con la línea de paso entre ambas líneas
IF mostrarPend THEN
    txtPend = STR("%.2", pend) + " %"
    Linc = SQR(Lt * Lt + hMed * hMed)       ! longitud inclinada del tramo, sin descansos
    txtLong = STR("%.2", Linc) + " m"
    hTx = tamTexto * GLOB_SCALE / 1000      ! tamaño del texto en metros de modelo
    fnt = fuenteTexto
    IF fnt = "" THEN fnt = "Arial"
    DEFINE STYLE "rampaTxt" fnt, tamTexto, 5, 0
    SET STYLE "rampaTxt"
    PEN penTexto
    angT = 90                               ! base del texto hacia la derecha, igual en ambos tramos
    IF orientTexto = 0 THEN                 ! Legible: se voltea si el símbolo girado lo dejaría al revés
        tn = 90 + SYMB_ROTANGLE
        tn = tn - 360 * INT(tn / 360)
        IF tn < 0 THEN tn = tn + 360
        IF tn > 90.5 AND tn < 269.5 THEN angT = 270
    ENDIF
    xT = xc1 : GOSUB "textoTramo"
    xT = xc2 : GOSUB "textoTramo"
ENDIF

END

! ---------- Subrutinas 2D ----------
"trSolido":
    LINE_TYPE 1
    LINE2 tx1, ty1, tx2, ty2
RETURN

"trDisc":
    IF ltSobre > 0 THEN
        LINE_TYPE ltSobre
    ELSE
        LINE_TYPE 2
    ENDIF
    LINE2 tx1, ty1, tx2, ty2
RETURN

"trAuto":
    IF tz < hc THEN
        GOSUB "trSolido"
    ELSE
        GOSUB "trDisc"
    ENDIF
RETURN

! Cara superior del tramo respecto al corte:
! stT = 0 todo sobre el corte · 1 todo bajo el corte · 2 el corte lo atraviesa (scT = recorrido donde ocurre)
"estadoTramo":
    hq = hc - zbT
    IF hq <= 0.0001 THEN
        stT = 0
        scT = 0
    ELSE
        IF hq >= hMed - 0.0001 THEN
            stT = 1
            scT = LF
        ELSE
            stT = 2
            GOSUB "uCruce"
            scT = ucr
        ENDIF
    ENDIF
RETURN

"uCruce":
    ucr = LF
    hallado = 0
    FOR iq = 1 TO np - 1
        IF hallado = 0 THEN
            IF zT[iq + 1] >= hq - 0.00001 THEN
                ucr = uT[iq] + (hq - zT[iq]) / (zT[iq + 1] - zT[iq]) * (uT[iq + 1] - uT[iq])
                hallado = 1
            ENDIF
        ENDIF
    NEXT iq
RETURN

! La rotura es oblicua: el borde interior cambia más adelante que el exterior
"bordes":
    IF stT = 2 THEN
        sInter = MIN(LF, scT + dk)
        sExter = MAX(0, scT - dk)
    ELSE
        sInter = scT
        sExter = scT
    ENDIF
RETURN

! Borde longitudinal de un tramo: continuo de 0 a sT, discontinuo de sT a LF (en el sentido de marcha)
"ladoTramo":
    IF fid = 1 THEN
        ya = 0
        yb = sT
        yc2 = LF
    ELSE
        ya = LF
        yb = LF - sT
        yc2 = 0
    ENDIF
    tx1 = lx
    tx2 = lx
    IF sT > 0.0001 THEN
        ty1 = ya
        ty2 = yb
        GOSUB "trSolido"
    ENDIF
    IF sT < LF - 0.0001 THEN
        ty1 = yb
        ty2 = yc2
        GOSUB "trDisc"
    ENDIF
RETURN

! Línea de rotura: "Z" limpio medido a lo largo de la propia línea de corte
"rotura":
    eB = 0.12
    hB = wA / 2 + eB
    lenK = SQR(1 + kb * kb)
    dx = 1 / lenK
    dy = kb / lenK
    nx = -kb / lenK
    ny = 1 / lenK
    zw = MIN(0.25 * wA, 0.25)
    zs = 0.8 * zw
    PEN penRotura

    IF rotSimbolo THEN
        LINE_TYPE ltRotura
        ra[1] = -hB * lenK : rd[1] = 0
        ra[2] = hB * lenK  : rd[2] = 0
        nPR = 2
    ELSE
        LINE_TYPE 1
        ra[1] = -hB * lenK : rd[1] = 0
        ra[2] = -zw        : rd[2] = 0
        ra[3] = -0.3 * zw  : rd[3] = zs
        ra[4] = 0.3 * zw   : rd[4] = -zs
        ra[5] = zw         : rd[5] = 0
        ra[6] = hB * lenK  : rd[6] = 0
        nPR = 6
    ENDIF

    FOR iq = 1 TO nPR
        xa = xcB + sgB * (ra[iq] * dx + rd[iq] * nx)
        sa = scB + ra[iq] * dy + rd[iq] * ny
        IF fidB = 1 THEN
            ya = sa
        ELSE
            ya = LF - sa
        ENDIF
        PUT xa, ya, 1
    NEXT iq
    POLY2_ NSP / 3, 1, GET(NSP)
    PEN gs_cont_pen
RETURN

! Texto de un tramo: pendiente a un lado y longitud al otro lado de la línea de paso
"textoTramo":
    ADD2 xT, LF / 2
    ROT2 angT
    TEXT2 0, 0.75 * hTx, txtPend
    TEXT2 0, -0.75 * hTx, txtLong
    DEL 2
RETURN
```

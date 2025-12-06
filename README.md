# Base de datos · Velagas

Documentación de los **nodos de SQL Server** usados en este flujo y los nodos auxiliares que preparan/consumen datos relacionados.

## Credenciales MSSQL

- microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account

## Resumen de nodos SQL

| Nodo | Operación | Tablas | Entradas | Salidas |
|---|---|---|---|---|
| Busqueda de cliente por numero de telefono2 | SELECT | Address, Customers | Datos relevantes | Validación de cliente1 |
| Actualizar cliente | INSERT | Address | datos direccion1 | Crear orden1 |
| Crear orden | INSERT | OrderDetails, Orders | preparar campos | Preparar resultado |
| Crear orden1 | INSERT | OrderDetails, Orders | Actualizar cliente | enviar sms |
| busqueda de colonia y ruta 1 | SELECT | ColRut, Colonias, Rutas | Selector | preparar campos |
| crear cliente | INSERT | Customers | Preparar consulta sql | crear direccion |
| crear detalle | INSERT | OrderDetails | crear orden | HTTP Request1 |
| crear direccion | INSERT | Address | crear cliente | crear orden |
| crear orden | INSERT | Orders | crear direccion | crear detalle |
| obtener colonia y ruta | SELECT | ColRut, Colonias, Rutas | Selector | datos direccion completos |
| obtener colonia y ruta 2 | SELECT | ColRut, Colonias, Rutas | Selector | datos direccion1 |

## Detalle por nodo SQL

### Busqueda de cliente por numero de telefono2
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `SELECT`
- **Tablas detectadas:** Address, Customers
- **Entradas:** Datos relevantes
- **Salidas:** Validación de cliente1
```sql
SELECT 
  c.CustomerId,
  c.Cellphone,
  c.FirstName,
  c.LastName1,
  c.LastName2,
  c.CustomerType,
  a.AddressId,
  a.AddressName,
  a.Street,
  a.ExtNumber,
  a.IntNumber,
  a.Reference,
  a.ZipCode,
  a.BetweenStreets1,
  a.BetweenStreets2,
  a.Location,
  a.Neighborghood
FROM 
  (SELECT '{{ $json.numero_cliente }}' AS InputCellphone) param
LEFT JOIN Customers c ON c.Cellphone = param.InputCellphone
LEFT JOIN Address a ON c.CustomerId = a.CustomerId;
```

### Actualizar cliente
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** Address
- **Entradas:** datos direccion1
- **Salidas:** Crear orden1
```sql
=INSERT INTO Address (
    AddressName,
    Street,
    ExtNumber,
    IntNumber,
    Reference,
    BetweenStreets1,
    BetweenStreets2,
    Neighborghood,
    ZipCode,
    Municipality,
    Location,
    State,
    CustomerId,
    CustomerTypeId,
    UbietyId
) 
  OUTPUT INSERTED.AddressId
  VALUES (
     '{{ $json.AddressName }}',
     '{{ $json.calle }}',
     '{{ $json.numExterior }}',
     '{{ $json.numInterior }}',
     '{{ $json.referencias }}',
     '{{ $json.entreCalle1 }}',
     '{{ $json.entreCalle2 }}',
     '{{ $json.colonia }}',
     '{{ $json.codPostal}}',
     '{{ $json.municipio }}',
     '{{ $json.municipio }}',
     '{{ $json.estado }}',
      {{ $('Selector').item.json.cliente_id }},
      1,
      1
  );
```

### Crear orden
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** OrderDetails, Orders
- **Entradas:** preparar campos
- **Salidas:** Preparar resultado
```sql
-- 1) Crear el pedido
INSERT INTO Orders (
    AddressId,
    CustomerId,
    PaymentMethodId,
    Source,
    Status,
    CreatedDate,
    RequiresInvoice,
    UserId,
    DeliveryDate
)
OUTPUT INSERTED.OrderId
VALUES (
    {{ $json.direccion_id }},
    {{ $json.cliente_id }},
    {{ $json.metodo_id }},
    'WEBHOOK',
    1,
    GETDATE(),
    0,
    '33463BD0-7F7C-408F-BD35-00DBB8A1A17A',
    {{ $json.deliverySqlValue }}
);

-- 2) Guardar el ID del pedido creado
DECLARE @OrderId BIGINT;
SET @OrderId = CAST(SCOPE_IDENTITY() AS BIGINT);

-- 3) Insertar detalle del pedido
INSERT INTO OrderDetails (
    OrderId,
    Pieces,
    Quantity,
    Subtotal,
    Service,
    CreatedDate,
    PriceId,
    RouteId,
    UnitMeasureId,
    IsSended
)
VALUES (
    @OrderId,
    {{ $json.numero_tanques || 1 }},
    {{
  (()=>{
    const t = ($json.tipo_producto||'')
      .toString().toLowerCase()
      .normalize('NFD').replace(/\p{Diacritic}/gu,'').trim();

    return (t === 'estacionario')
      ? Number($('datos webhook').first().json.body.message.toolCalls[0].function.arguments.numero_litros)
      : Number((($json.tipo_producto||'').toString().match(/\d+/)||[])[0]);
  })()
}},
    {{ $json.precio_total }},
    '{{ $json.tipo_producto }}',
    GETDATE(),
    1,
    {{ $json.ruta_id || 0}},
    1,
    0
);
```

### Crear orden1
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** OrderDetails, Orders
- **Entradas:** Actualizar cliente
- **Salidas:** enviar sms
```sql
-- 1) Crear el pedido
INSERT INTO Orders (
    AddressId,
    CustomerId,
    PaymentMethodId,
    Source,
    Status,
    CreatedDate,
    RequiresInvoice,
    UserId,
    DeliveryDate
)
OUTPUT INSERTED.OrderId
VALUES (
    {{ $json.AddressId }},
    {{ $('Selector').item.json.cliente_id }},
    {{
  (()=>{
    const val =
      $json.forma_pago ??
      $('Datos relevantes').first().json.forma_pago ??
      $('Validación de cliente1').first().json.datos_pedido?.forma_pago ??
      $('Selector').first().json.datos_pedido?.forma_pago ?? '';

    const clean = String(val)
      .toLowerCase()
      .normalize('NFD').replace(/\p{Diacritic}/gu,'')  // quita acentos
      .trim();                                         // quita espacios

    return clean === 'efectivo' ? 2 : 3;
  })()
}}
,
    'WEBHOOK',
    1,
    GETDATE(),
    0,
    '33463BD0-7F7C-408F-BD35-00DBB8A1A17A',
    {{
  (()=>{
    const s = $('Selector').first().json.horario_entrega;
    return s
      ? `CASE WHEN ISDATE('${s}')=1 THEN CONVERT(datetime,'${s}',103) ELSE NULL END`
      : 'NULL';
  })()
}}
);

-- 2) Guardar el ID del pedido creado
DECLARE @OrderId BIGINT;
SET @OrderId = CAST(SCOPE_IDENTITY() AS BIGINT);

-- 3) Insertar detalle del pedido
INSERT INTO OrderDetails (
    OrderId,
    Pieces,
    Quantity,
    Subtotal,
    Service,
    CreatedDate,
    PriceId,
    RouteId,
    UnitMeasureId,
    IsSended,
    Status
)
VALUES (
    @OrderId,
    {{ $('Selector').item.json.datos_pedido.numero_tanques || 1}},
      
  {{
  (()=>{
    // 1) Tomar tipo_producto del nodo correcto (prioriza "Selector", luego "Datos relevantes", luego el item actual)
    const tipo = (
      $('Selector').first().json.datos_pedido?.tipo_producto ??
      $('Datos relevantes').first().json.tipo_producto ??
      $json.tipo_producto ?? ''
    ).toString();

    const clean = tipo.toLowerCase().normalize('NFD').replace(/\p{Diacritic}/gu,'').trim();

    // 2) Si es estacionario, tomar numero_litros (hay 2 posibles rutas en tu payload)
    if (clean.startsWith('estacionario')) {
      const ts = $('Tool Selector').first().json;
      const litros = Number(
        ts?.body?.message?.toolCalls?.[0]?.function?.arguments?.numero_litros ??
        ts?.toolCallList?.[0]?.function?.arguments?.numero_litros ??
        ''
      );
      return Number.isFinite(litros) ? litros : 0; // pon 0 o NULL si prefieres
    }

    // 3) Si es cilindro, extrae el número (10, 20, 30, 45, 300...)
    const m = tipo.match(/\d+/);
    const n = m ? Number(m[0]) : 0;              // pon 0 o NULL si prefieres
    return Number.isFinite(n) ? n : 0;
  })()
}}

  
  ,
    {{ $('Selector').item.json.datos_pedido.precio_total }},
    '{{ $('Selector').item.json.datos_pedido.tipo_producto }}',
    GETDATE(),
    1,
    {{ $('datos direccion1').item.json.RouteId || 0 }},
    1,
    0,
    1
);
```

### busqueda de colonia y ruta 1
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `SELECT`
- **Tablas detectadas:** ColRut, Colonias, Rutas
- **Entradas:** -
- **Salidas:** preparar campos
```sql
USE dbintergas;

SELECT TOP 3
    r.RouteId,
    r.RouteName,
    cr.ColoniasId
FROM Rutas r
INNER JOIN ColRut cr ON r.RouteId = cr.RouteId
INNER JOIN Colonias col ON cr.ColoniasId = col.ColoniaId
WHERE col.Nombre LIKE '%{{ $json.datos_direccion_seleccionada.colonia }}%'
ORDER BY 
    CASE 
        WHEN col.Nombre = '{{ $json.datos_direccion_seleccionada.colonia }}' THEN 1
        ELSE 2
    END;
```

### crear cliente
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** - Customers
- **Entradas:** Preparar consulta sql
- **Salidas:** crear direccion
```sql
INSERT INTO Customers (
    Phone,
    FirstName,
    Cellphone,
    Origen,
    CustomerType,
    CreatedDate,
    CreatedUserId,
    RequiresValidation,
    LastName1,
    LastName2
)
OUTPUT INSERTED.CustomerId INTO @NewCustomers(CustomerId)
VALUES (
    ${telefono_limpio ?? 'NULL'},
    '${nombre_cliente}',
    '${telefono_raw}',
    '${source}',
    'PARTICULAR',
    GETDATE(),
    '${CONFIG.validUserId}',
    0,
    '',
    ''
);

SELECT CustomerId FROM @NewCustomers;  
COMMIT TRAN;
```

### crear direccion
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** - Address
- **Entradas:** crear cliente
- **Salidas:** crear orden
```sql
USE dbintergas;
BEGIN TRAN;

DECLARE @NewAddress TABLE (AddressId INT);

INSERT INTO Address (
    CustomerId,
    CustomerTypeId,
    UbietyId,
    ColoniaId,
    AddressName,
    Street,
    ExtNumber,
    ZipCode,
    [Reference],
    RequiresValidation,
    IntNumber,
    BetweenStreets1,
    BetweenStreets2,
    Neighborghood,
    State,
    Location,
    Municipality
)
OUTPUT INSERTED.AddressId INTO @NewAddress(AddressId)
VALUES (
    {CustomerId},                 
    ${CONFIG.customerTypeId},
    ${CONFIG.ubietyId},
    ${coloniaIdFinal},
    'Dirección Principal',
    '${street}',
    '${extNum}',
    '${zipCode}',
    '${reference}',
    0,
    '${intNum}',
    '${entre_calle1}',
    '${entre_calle2}',
    '${colonia}',
    '${estado}',
    '${municipio}',
    '${municipio}'
);

SELECT AddressId FROM @NewAddress;
COMMIT TRAN;
```

### crear orden
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** - Orders
- **Entradas:** crear direccion
- **Salidas:** crear detalle
```sql
USE dbintergas;
BEGIN TRAN;

DECLARE @NewOrders TABLE (OrderId INT);

INSERT INTO Orders (
  AddressId, 
  CustomerId, 
  PaymentMethodId, 
  Source, 
  RequiresInvoice, 
  Status, 
  CreatedDate, 
  UserId, 
  RequiresValidation, 
  Scheduled, 
  CustomerDelivery,
  FechaCreacion,
  DeliveryDate
)
OUTPUT INSERTED.OrderId INTO @NewOrders(OrderId)
VALUES (
  {AddressId},                    
  {CustomerId},                   
  ${CONFIG.paymentMethodId},
  '${source}',
  0,
  1,
  GETDATE(),
  '${CONFIG.validUserId}',
  0,
  0,
  0,
  GETDATE(),
  ${deliverySqlValue}
);

SELECT OrderId FROM @NewOrders;   -- <- devuelve el ID
COMMIT TRAN;
```

### crear detalle
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `INSERT`
- **Tablas detectadas:** - OrderDetails
- **Entradas:** crear orden
- **Salidas:** HTTP Request1
```sql
USE dbintergas;
BEGIN TRAN;

DECLARE @NewOrderDetails TABLE (OrderDetailId INT);

INSERT INTO OrderDetails (
  OrderId,
  Pieces,
  UnitMeasureId,
  Quantity,
  Subtotal,
  Service,
  CreatedDate,
  PriceId,
  RouteId,
  Status,
  PrecioId,
  IsSended
)
OUTPUT INSERTED.OrderDetailId INTO @NewOrderDetails(OrderDetailId)
VALUES (
  {OrderId},
  ${numero_tanques},
  ${CONFIG.unitMeasureId},
  ${cantidad_kilos},
  ${precio_total},
  '${tipo_producto}',
  GETDATE(),
  ${CONFIG.priceId},
  ${routeIdToUse},
  1,
  ${CONFIG.precioId},
  0
);

SELECT OrderDetailId FROM @NewOrderDetails;  -- <- devuelve el ID
COMMIT TRAN;
```


### obtener colonia y ruta
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `SELECT`
- **Tablas detectadas:** ColRut, Colonias, Rutas
- **Entradas:** Selector
- **Salidas:** datos direccion completos
```sql
USE dbintergas;

SELECT TOP 3
    r.RouteId,
    r.RouteName,
    cr.ColoniasId
FROM Rutas r
INNER JOIN ColRut cr ON r.RouteId = cr.RouteId
INNER JOIN Colonias col ON cr.ColoniasId = col.ColoniaId
WHERE col.Nombre LIKE '%{{ $json.direccion_referencia.colonia }}%'
ORDER BY 
    CASE 
        WHEN col.Nombre = '{{ $json.direccion_referencia.colonia }}' THEN 1
        ELSE 2
    END;
```

### obtener colonia y ruta 2
- **Tipo:** `n8n-nodes-base.microsoftSql`
- **Credenciales:** microsoftSql: id=jpBzkdyAr4CclqkP, name=Microsoft SQL account
- **Operación:** `SELECT`
- **Tablas detectadas:** ColRut, Colonias, Rutas
- **Entradas:** -
- **Salidas:** datos direccion1
```sql
USE dbintergas;

SELECT TOP 3
    r.RouteId,
    r.RouteName,
    cr.ColoniasId
FROM Rutas r
INNER JOIN ColRut cr ON r.RouteId = cr.RouteId
INNER JOIN Colonias col ON cr.ColoniasId = col.ColoniaId
WHERE col.Nombre LIKE '{{ $json.direccion_referencia.colonia }}'
ORDER BY 
    CASE 
        WHEN col.Nombre = '{{ $json.direccion_referencia.colonia }}' THEN 1
        ELSE 2
    END;
```

## Checklist de despliegue

- Configurar las credenciales MSSQL anteriores en n8n.
- Verificar permisos en SQL Server según las operaciones (SELECT/INSERT/UPDATE/DELETE).
- Comprobar que los nodos auxiliares generan los campos requeridos por cada consulta.
- Probar el flujo con datos controlados antes de producción.

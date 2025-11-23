# LaboratoriosUDLA

🚨 CERO: Antes de empezar (Seguridad Financiera)
Para evitar sorpresas en tu tarjeta de crédito:

Ve a Billing Dashboard > Budgets.

Crea un presupuesto (Budget) de tipo "Cost budget".

Ponle un límite de $1.00 USD.

Configura una alerta a tu email cuando llegue al 80% ($0.80).

Esto te avisará si algo se sale de control antes de que sea grave.

PASO 1: Base de Datos (DynamoDB)
Igual que antes, pero asegurando que esté en la capa gratuita.

Ve a DynamoDB > Create table.

Table name: ReservasUDLA.

Partition key: SalaID (String).

Sort key: FechaHora (String).

Table settings: Elige "Customize settings".

Capacity mode: Provisioned (Importante: la capa gratuita te da 25 RCU y 25 WCU. Si eliges "On-Demand" te podrían cobrar si el tráfico es masivo, aunque para pruebas es ínfimo).

Asegúrate de que "Auto scaling" esté activado.

Create table.

PASO 2: Seguridad y Permisos (IAM) - NUEVO
En el Free Tier no existe el LabRole. Debemos crear un rol para que tu Lambda pueda tocar la base de datos.

Ve a IAM > Roles > Create role.

Trusted entity type: AWS Service.

Service or use case: Lambda.

Add permissions: Busca y marca estas dos casillas:

AmazonDynamoDBFullAccess (Para leer/escribir en la base).

AWSLambdaBasicExecutionRole (Para poder escribir logs y ver errores).

Role name: RolServidorReservas.

Create role.

PASO 3: Autenticación (Cognito)
La capa gratuita de Cognito permite 50,000 usuarios activos mensuales gratis. Estás sobrado.

Ve a Cognito > Create user pool.

Sign-in options: Email.

Password policy: Relaja los requisitos si quieres probar rápido (ej: quita símbolos). MFA: No MFA.

User account recovery: Deja por defecto.

App integration:

Pool name: UDLAPool.

Domain: Crea un dominio (ej: reservas-proyecto-tuapellido).

App client: Public client. Name: WebClient.

Client Secret: Selecciona "Don't generate a client secret" (CRUCIAL para que funcione en web).

Allowed callback URLs: Pon la URL de tu S3 (cuando la tengas) y http://localhost para pruebas.

Create.

Usuarios y Grupos:

Entra al Pool > Groups > Crea Administradores y Estudiantes.

Crea un usuario (Create user) y asígnalo a Administradores.

PASO 4: Backend (Lambda)
Ve a Lambda > Create function.

Name: BackendReservas.

Runtime: Python 3.9+.

Change default execution role:

Selecciona "Use an existing role".

Elige el RolServidorReservas que creaste en el Paso 2.

Code: Pega el siguiente código (adaptado para Free Tier y Cognito):

Python

import json
import boto3
from botocore.exceptions import ClientError

# Inicializar DynamoDB
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('ReservasUDLA')

def lambda_handler(event, context):
    print("Evento recibido:", json.dumps(event)) # Esto aparecerá en CloudWatch Logs
    
    # CORS Headers (Necesarios para que el navegador no bloquee)
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token',
        'Access-Control-Allow-Methods': 'OPTIONS,POST,DELETE'
    }

    # Manejo de OPTIONS (Pre-flight request del navegador)
    if event['httpMethod'] == 'OPTIONS':
        return {'statusCode': 200, 'headers': headers, 'body': ''}

    # Datos del usuario desde Cognito
    try:
        claims = event['requestContext']['authorizer']['claims']
        email = claims['email']
        grupos = claims.get('cognito:groups', '')
    except (KeyError, TypeError):
        # Fallback solo si pruebas desde la consola de AWS (Test button)
        email = "admin-test@udla.edu.ec"
        grupos = "Administradores"

    body = json.loads(event.get('body', '{}'))

    # --- BORRAR RESERVA (SOLO ADMIN) ---
    if event['httpMethod'] == 'DELETE':
        if 'Administradores' not in grupos:
            return {'statusCode': 403, 'headers': headers, 'body': json.dumps({'error': 'Solo admins pueden borrar'})}
        
        try:
            table.delete_item(Key={'SalaID': body['sala_id'], 'FechaHora': body['fecha_hora']})
            return {'statusCode': 200, 'headers': headers, 'body': json.dumps({'mensaje': 'Reserva eliminada'})}
        except Exception as e:
            return {'statusCode': 500, 'headers': headers, 'body': json.dumps({'error': str(e)})}

    # --- CREAR RESERVA (TODOS) ---
    elif event['httpMethod'] == 'POST':
        try:
            table.put_item(
                Item={
                    'SalaID': body['sala_id'],
                    'FechaHora': body['fecha_hora'],
                    'Usuario': email,
                    'Estado': 'Reservado'
                },
                # Evitar doble reserva
                ConditionExpression='attribute_not_exists(SalaID)'
            )
            return {'statusCode': 200, 'headers': headers, 'body': json.dumps({'mensaje': 'Reserva exitosa'})}
        except ClientError as e:
            if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
                return {'statusCode': 409, 'headers': headers, 'body': json.dumps({'error': 'Horario ocupado'})}
            else:
                return {'statusCode': 500, 'headers': headers, 'body': json.dumps({'error': str(e)})}

    return {'statusCode': 400, 'headers': headers, 'body': json.dumps({'error': 'Método no soportado'})}
Deploy.

PASO 5: API Gateway
La capa gratuita te da 1 millón de llamadas al mes.

Ve a API Gateway > REST API (Build).

Name: API_Reservas.

Authorizers (Menú izquierdo):

Create New. Name: CognitoAuth.

Type: Cognito.

User Pool: UDLAPool.

Token Source: Authorization.

Create.

Resources:

Create Resource: /reservar. Check Enable API Gateway CORS (Esto te ahorra muchos dolores de cabeza).

Methods:

En /reservar, crea método POST. Integration: Lambda (BackendReservas). Check "Lambda Proxy Integration".

En /reservar, crea método DELETE. Integration: Lambda (BackendReservas). Check "Lambda Proxy Integration".

Proteger Métodos:

Entra al Method Request del POST. Authorization: CognitoAuth.

Entra al Method Request del DELETE. Authorization: CognitoAuth.

Deploy API:

Stage: prod.

Copia la "Invoke URL".

PASO 6: Frontend (S3)
Capa gratuita: 5GB de almacenamiento y 20,000 GET requests.

Crea un archivo index.html en tu PC.

Pega el código de abajo (es una versión mejorada y más limpia).

Ve a S3 > Create bucket.

Name: udla-reserva-app-tunombre.

Desmarca "Block all public access". Confirma la advertencia.

Sube el index.html.

Ve a Properties > Static website hosting > Enable.

Ve a Permissions > Bucket Policy:

JSON

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::udla-reserva-app-tunombre/*"
        }
    ]
}
(Cambia el nombre del bucket en el JSON).

CÓDIGO FINAL (index.html)
Importante: Debes reemplazar las constantes al inicio del script.

HTML

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>UDLA Reservas Cloud</title>
    <style>
        /* Estilo simple y limpio tipo UDLA */
        body { font-family: 'Segoe UI', sans-serif; background: #f8f9fa; color: #333; display: flex; justify-content: center; align-items: center; min-height: 100vh; margin: 0; }
        .card { background: white; padding: 2rem; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); width: 100%; max-width: 400px; }
        h2 { color: #c8102e; text-align: center; margin-bottom: 1.5rem; }
        .btn { width: 100%; padding: 12px; border: none; border-radius: 5px; cursor: pointer; font-size: 16px; margin-top: 10px; transition: 0.3s; }
        .btn-primary { background: #c8102e; color: white; }
        .btn-primary:hover { background: #a00c24; }
        .btn-danger { background: #333; color: white; }
        input, select { width: 100%; padding: 10px; margin: 5px 0 15px 0; border: 1px solid #ddd; border-radius: 5px; box-sizing: border-box; }
        .hidden { display: none; }
        #status { text-align: center; margin-top: 15px; font-weight: bold; }
    </style>
</head>
<body>

<div class="card">
    <div style="text-align:center;">
        <img src="https://upload.wikimedia.org/wikipedia/commons/6/68/Logotipo_UDLA.png" width="100" alt="Logo">
    </div>
    <h2>Reservas Cloud</h2>

    <div id="loginSection">
        <p style="text-align:center;">Ingresa con tus credenciales institucionales</p>
        <button class="btn btn-primary" onclick="login()">Iniciar Sesión</button>
    </div>

    <div id="appSection" class="hidden">
        <p>Hola, <b id="userEmail">Usuario</b></p>
        <div id="adminBadge" class="hidden" style="background:#ffd700; color:#000; padding:2px 5px; font-size:12px; border-radius:3px; display:inline-block; margin-bottom:10px;">ADMINISTRADOR</div>
        
        <label>Laboratorio / Sala</label>
        <select id="salaId">
            <option value="LAB_CIBER_1">Lab Ciberseguridad 1</option>
            <option value="LAB_REDES_2">Lab Redes 2</option>
            <option value="AUDITORIO">Auditorio Principal</option>
        </select>

        <label>Fecha y Hora (Formato ID)</label>
        <input type="text" id="fechaHora" placeholder="2025-11-25-0900">

        <button class="btn btn-primary" onclick="enviar('POST')">Reservar</button>
        
        <div id="adminControls" class="hidden">
            <hr>
            <label style="color:red; font-size:12px;">ZONA ADMINISTRATIVA</label>
            <button class="btn btn-danger" onclick="enviar('DELETE')">Eliminar Reserva</button>
        </div>
        
        <p id="status"></p>
        <a href="#" onclick="logout()" style="display:block; text-align:center; margin-top:20px; color:#666;">Cerrar Sesión</a>
    </div>
</div>

<script>
    // --- ⚠️ REEMPLAZA ESTOS DATOS ⚠️ ---
    // 1. Tu URL de API Gateway (Termina en /prod/reservar)
    const API_URL = 'https://XXXXXXXX.execute-api.us-east-1.amazonaws.com/prod/reservar';
    // 2. Dominio de Cognito (Sin el /login al final)
    const COGNITO_DOMAIN = 'https://tudominio.auth.us-east-1.amazoncognito.com';
    // 3. Client ID de Cognito (App integration > App client list)
    const CLIENT_ID = 'xxxxxxxxxxxxxxxxxxxxxx';
    // 4. URL de este sitio S3 (tal cual la pusiste en Cognito Callback URL)
    const REDIRECT_URI = 'http://tu-bucket.s3-website-us-east-1.amazonaws.com';
    
    let idToken = null;

    window.onload = () => {
        // Verificar si volvimos del login con el token en la URL
        const urlParams = new URLSearchParams(window.location.hash.replace('#', '?'));
        idToken = urlParams.get('id_token');

        if (idToken) {
            mostrarApp(idToken);
            // Limpiar URL para que se vea bonita
            window.history.replaceState({}, document.title, window.location.pathname);
        }
    };

    function login() {
        // Redirigir a la UI de Cognito
        window.location.href = `${COGNITO_DOMAIN}/login?client_id=${CLIENT_ID}&response_type=token&scope=email+openid&redirect_uri=${REDIRECT_URI}`;
    }

    function logout() {
        window.location.href = `${COGNITO_DOMAIN}/logout?client_id=${CLIENT_ID}&logout_uri=${REDIRECT_URI}`;
    }

    function mostrarApp(token) {
        document.getElementById('loginSection').classList.add('hidden');
        document.getElementById('appSection').classList.remove('hidden');

        // Decodificar JWT para sacar info básica (sin validar firma, solo visual)
        const payload = JSON.parse(atob(token.split('.')[1]));
        document.getElementById('userEmail').innerText = payload.email;

        // Verificar grupo admin
        if(payload['cognito:groups'] && payload['cognito:groups'].includes('Administradores')){
            document.getElementById('adminBadge').classList.remove('hidden');
            document.getElementById('adminControls').classList.remove('hidden');
        }
    }

    async function enviar(metodo) {
        const sala = document.getElementById('salaId').value;
        const fecha = document.getElementById('fechaHora').value;
        const status = document.getElementById('status');

        status.innerText = "Procesando...";
        status.style.color = "blue";

        try {
            const res = await fetch(API_URL, {
                method: metodo,
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': idToken // El token es la llave
                },
                body: JSON.stringify({ sala_id: sala, fecha_hora: fecha })
            });

            const data = await res.json();

            if (res.ok) {
                status.innerText = data.mensaje || "Éxito";
                status.style.color = "green";
            } else {
                status.innerText = data.error || "Error desconocido";
                status.style.color = "red";
            }
        } catch (e) {
            status.innerText = "Error de conexión";
            status.style.color = "red";
        }
    }
</script>
</body>
</html>
⚠️ Consideraciones Finales del Free Tier
Callbacks: Si cuando le das "Iniciar Sesión" te sale un error de Cognito, es 100% seguro que la URL en Allowed callback URLs en Cognito no coincide letra por letra con la URL de tu S3 (cuidado con las barras / al final).

Región: Haz TODO en us-east-1 (N. Virginia). Es la más barata y donde siempre están las funcionalidades completas.

Limpieza: Cuando termines el proyecto y te pongan la nota:

Borra el Bucket S3.

Borra la tabla DynamoDB.

Borra el API Gateway.

(Opcional) Borra el User Pool y Lambda.

Lo más caro si te olvidas: No estamos usando NAT Gateway ni EC2, así que tu riesgo es bajo, pero borrar todo es buena práctica.

---
title: "gRPC ile Yüksek Performanslı API'ler"
description: "gRPC ve Protocol Buffers ile yüksek performanslı API geliştirme. Python implementasyonu, streaming, interceptors, load balancing ve microservice best practices."
date: "2025-10-22 09:00:00 +0300"
categories: [Backend, API]
tags: [grpc, protobuf, microservices, api, python, performance, rest, rpc, streaming, http2]
image:
  path: /assets/img/posts/grpc-architecture-workflow.png
  alt: "gRPC Architecture ve Workflow Diyagramı"
---

Modern mikroservis mimarilerinde servisler arası iletişim kritik bir rol oynar. REST API'ler uzun yıllardır standart olsa da, gRPC (Google Remote Procedure Call) yüksek performans, düşük latency ve güçlü type-safety özellikleriyle öne çıkıyor. Bu yazıda, gRPC'nin temellerini, Protocol Buffers'ı, Python ile implementasyonunu ve best practice'leri ele alacağız.

## gRPC Nedir ve Neden Önemli?

gRPC, Google tarafından geliştirilen ve HTTP/2 üzerinde çalışan modern bir RPC (Remote Procedure Call) framework'üdür. REST API'lere alternatif olarak, özellikle mikroservis mimarilerinde tercih edilir.

### gRPC'nin Temel Avantajları

- **Yüksek Performans**: Binary serialization (Protocol Buffers) sayesinde JSON'dan 5-10x daha hızlı
- **HTTP/2**: Multiplexing, server push, header compression gibi özellikler
- **Streaming**: Unary, server streaming, client streaming, bidirectional streaming desteği
- **Type-Safety**: Protocol Buffers ile güçlü tip kontrolü
- **Code Generation**: Otomatik client ve server kodu üretimi
- **Çoklu Dil Desteği**: 10+ programlama dili için resmi destek

![gRPC vs REST Performans Karşılaştırması](/assets/img/posts/grpc-vs-rest-performance-comparison.png)

### gRPC vs REST: Karşılaştırma

| Özellik | gRPC | REST |
|---------|------|------|
| **Protocol** | HTTP/2 | HTTP/1.1 |
| **Payload Format** | Protocol Buffers (binary) | JSON (text) |
| **Streaming** | Bidirectional | Sınırlı |
| **Performance** | Yüksek | Orta |
| **Browser Support** | Sınırlı (gRPC-Web gerekli) | Tam |
| **Human Readable** | Hayır | Evet |
| **Code Generation** | Otomatik | Manuel/3rd party |

## Protocol Buffers: gRPC'nin Kalbindeki Teknoloji

Protocol Buffers (protobuf), Google'ın geliştirdiği language-agnostic binary serialization formatıdır.

![Protocol Buffers ve gRPC İlişkisi](/assets/img/posts/grpc-protobuf-concept.png)

### .proto Dosyası Tanımlama

```protobuf
// user.proto
syntax = "proto3";

package user;

// User message tanımı
message User {
  int32 id = 1;
  string username = 2;
  string email = 3;
  bool is_active = 4;
  repeated string roles = 5;  // repeated = list/array
  google.protobuf.Timestamp created_at = 6;
}

// Request/Response mesajları
message GetUserRequest {
  int32 user_id = 1;
}

message GetUserResponse {
  User user = 1;
  string message = 2;
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
}

message CreateUserResponse {
  User user = 1;
  bool success = 2;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  int32 page = 3;
}

// Service tanımı
service UserService {
  // Unary RPC - tek request, tek response
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  
  // Unary RPC - kullanıcı oluşturma
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  
  // Server streaming - sunucu birden fazla response gönderir
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming - client birden fazla request gönderir
  rpc BulkCreateUsers(stream CreateUserRequest) returns (CreateUserResponse);
  
  // Bidirectional streaming - hem client hem server stream
  rpc ChatWithSupport(stream ChatMessage) returns (stream ChatMessage);
}

// Chat mesajı için ek message
message ChatMessage {
  int32 user_id = 1;
  string message = 2;
  google.protobuf.Timestamp timestamp = 3;
}
```

### Protocol Buffers Data Types

```protobuf
syntax = "proto3";

message DataTypes {
  // Sayılar
  int32 age = 1;           // 32-bit integer
  int64 big_number = 2;    // 64-bit integer
  uint32 count = 3;        // unsigned 32-bit
  float price = 4;         // 32-bit float
  double precise = 5;      // 64-bit double
  
  // String ve bytes
  string name = 6;         // UTF-8 string
  bytes data = 7;          // arbitrary bytes
  
  // Boolean
  bool is_active = 8;
  
  // Enum
  Status status = 9;
  
  // Nested message
  Address address = 10;
  
  // Repeated (list/array)
  repeated string tags = 11;
  
  // Map
  map<string, int32> scores = 12;
  
  // Oneof - birden fazla alan aynı anda kullanılamaz
  oneof payment_method {
    CreditCard credit_card = 13;
    PayPal paypal = 14;
  }
}

enum Status {
  UNKNOWN = 0;  // Default değer 0 olmalı
  ACTIVE = 1;
  INACTIVE = 2;
  SUSPENDED = 3;
}

message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string zip_code = 4;
}

message CreditCard {
  string number = 1;
  string cvv = 2;
}

message PayPal {
  string email = 1;
}
```

## Python ile gRPC Server Implementasyonu

### Kurulum ve Proto Derleme

```bash
# gRPC ve protobuf kurulumu
pip install grpcio grpcio-tools

# Proto dosyasını derle (Python kodu oluştur)
python -m grpc_tools.protoc \
    -I. \
    --python_out=. \
    --grpc_python_out=. \
    user.proto

# Oluşturulan dosyalar:
# user_pb2.py - message tanımları
# user_pb2_grpc.py - service tanımları
```

### gRPC Server Implementation

```python
# server.py
import grpc
from concurrent import futures
import time
from datetime import datetime
import user_pb2
import user_pb2_grpc
from google.protobuf.timestamp_pb2 import Timestamp

# In-memory veritabanı (örnek için)
users_db = {}
user_id_counter = 1

class UserServiceServicer(user_pb2_grpc.UserServiceServicer):
    """UserService implementasyonu"""
    
    def GetUser(self, request, context):
        """Tek kullanıcı getir - Unary RPC"""
        user_id = request.user_id
        
        if user_id not in users_db:
            # gRPC error döndür
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f'User with ID {user_id} not found')
            return user_pb2.GetUserResponse()
        
        user = users_db[user_id]
        return user_pb2.GetUserResponse(
            user=user,
            message="User retrieved successfully"
        )
    
    def CreateUser(self, request, context):
        """Kullanıcı oluştur - Unary RPC"""
        global user_id_counter
        
        # Validasyon
        if not request.username or not request.email:
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            context.set_details('Username and email are required')
            return user_pb2.CreateUserResponse(success=False)
        
        # Email kontrolü
        for user in users_db.values():
            if user.email == request.email:
                context.set_code(grpc.StatusCode.ALREADY_EXISTS)
                context.set_details('Email already exists')
                return user_pb2.CreateUserResponse(success=False)
        
        # Timestamp oluştur
        timestamp = Timestamp()
        timestamp.GetCurrentTime()
        
        # User oluştur
        user = user_pb2.User(
            id=user_id_counter,
            username=request.username,
            email=request.email,
            is_active=True,
            roles=['user'],
            created_at=timestamp
        )
        
        users_db[user_id_counter] = user
        user_id_counter += 1
        
        return user_pb2.CreateUserResponse(
            user=user,
            success=True
        )
    
    def ListUsers(self, request, context):
        """Kullanıcı listesi - Server Streaming"""
        page = request.page if request.page > 0 else 1
        page_size = request.page_size if request.page_size > 0 else 10
        filter_text = request.filter.lower()
        
        # Filtreleme
        filtered_users = []
        for user in users_db.values():
            if not filter_text or \
               filter_text in user.username.lower() or \
               filter_text in user.email.lower():
                filtered_users.append(user)
        
        # Pagination
        start_idx = (page - 1) * page_size
        end_idx = start_idx + page_size
        page_users = filtered_users[start_idx:end_idx]
        
        # Stream olarak kullanıcıları gönder
        for user in page_users:
            yield user
            time.sleep(0.1)  # Simulated delay
    
    def BulkCreateUsers(self, request_iterator, context):
        """Toplu kullanıcı oluşturma - Client Streaming"""
        global user_id_counter
        created_count = 0
        last_user = None
        
        for create_request in request_iterator:
            # Her gelen request için kullanıcı oluştur
            timestamp = Timestamp()
            timestamp.GetCurrentTime()
            
            user = user_pb2.User(
                id=user_id_counter,
                username=create_request.username,
                email=create_request.email,
                is_active=True,
                roles=['user'],
                created_at=timestamp
            )
            
            users_db[user_id_counter] = user
            user_id_counter += 1
            created_count += 1
            last_user = user
        
        return user_pb2.CreateUserResponse(
            user=last_user,
            success=True
        )
    
    def ChatWithSupport(self, request_iterator, context):
        """Çift yönlü chat - Bidirectional Streaming"""
        for message in request_iterator:
            # Gelen mesajı al
            print(f"Received: {message.message} from user {message.user_id}")
            
            # Otomatik cevap gönder
            timestamp = Timestamp()
            timestamp.GetCurrentTime()
            
            response = user_pb2.ChatMessage(
                user_id=0,  # System user
                message=f"Echo: {message.message}",
                timestamp=timestamp
            )
            
            yield response
            time.sleep(0.5)  # Simulated processing time

def serve():
    """gRPC sunucusunu başlat"""
    # Thread pool oluştur
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    
    # Servicer'ı ekle
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(), server
    )
    
    # Port dinle
    port = '50051'
    server.add_insecure_port(f'[::]:{port}')
    
    print(f"gRPC Server starting on port {port}...")
    server.start()
    
    try:
        server.wait_for_termination()
    except KeyboardInterrupt:
        print("\nShutting down server...")
        server.stop(0)

if __name__ == '__main__':
    serve()
```

## gRPC Client Implementasyonu

```python
# client.py
import grpc
import user_pb2
import user_pb2_grpc
from google.protobuf.timestamp_pb2 import Timestamp

def run_unary_example(stub):
    """Unary RPC örneği"""
    print("\n=== Unary RPC Example ===")
    
    # Kullanıcı oluştur
    create_request = user_pb2.CreateUserRequest(
        username="john_doe",
        email="john@example.com",
        password="secure123"
    )
    
    try:
        response = stub.CreateUser(create_request)
        print(f"User created: {response.user.username} (ID: {response.user.id})")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")
        return None
    
    # Kullanıcıyı getir
    get_request = user_pb2.GetUserRequest(user_id=response.user.id)
    
    try:
        get_response = stub.GetUser(get_request)
        print(f"Retrieved user: {get_response.user.username}")
        print(f"Message: {get_response.message}")
        return get_response.user.id
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")
        return None

def run_server_streaming_example(stub):
    """Server Streaming örneği"""
    print("\n=== Server Streaming Example ===")
    
    request = user_pb2.ListUsersRequest(
        page=1,
        page_size=5,
        filter=""
    )
    
    try:
        # Server'dan stream olarak kullanıcıları al
        user_stream = stub.ListUsers(request)
        
        print("Receiving users stream:")
        for user in user_stream:
            print(f"  - {user.username} ({user.email})")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

def run_client_streaming_example(stub):
    """Client Streaming örneği"""
    print("\n=== Client Streaming Example ===")
    
    def generate_users():
        """Kullanıcı stream'i oluştur"""
        users_to_create = [
            {"username": "alice", "email": "alice@example.com", "password": "pass1"},
            {"username": "bob", "email": "bob@example.com", "password": "pass2"},
            {"username": "charlie", "email": "charlie@example.com", "password": "pass3"},
        ]
        
        for user_data in users_to_create:
            yield user_pb2.CreateUserRequest(**user_data)
            print(f"Sending: {user_data['username']}")
    
    try:
        response = stub.BulkCreateUsers(generate_users())
        print(f"Bulk creation completed. Last user: {response.user.username}")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

def run_bidirectional_streaming_example(stub):
    """Bidirectional Streaming örneği"""
    print("\n=== Bidirectional Streaming Example ===")
    
    def generate_messages():
        """Chat mesajları oluştur"""
        messages = ["Hello", "How are you?", "Need help with gRPC"]
        
        for msg in messages:
            timestamp = Timestamp()
            timestamp.GetCurrentTime()
            
            yield user_pb2.ChatMessage(
                user_id=1,
                message=msg,
                timestamp=timestamp
            )
            print(f"Sent: {msg}")
    
    try:
        # Bidirectional stream başlat
        responses = stub.ChatWithSupport(generate_messages())
        
        print("\nReceiving responses:")
        for response in responses:
            print(f"  Support: {response.message}")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

def main():
    """Ana client fonksiyonu"""
    # Channel oluştur (bağlantı)
    channel = grpc.insecure_channel('localhost:50051')
    
    # Stub (client) oluştur
    stub = user_pb2_grpc.UserServiceStub(channel)
    
    try:
        # Unary RPC
        run_unary_example(stub)
        
        # Server Streaming
        run_server_streaming_example(stub)
        
        # Client Streaming
        run_client_streaming_example(stub)
        
        # Bidirectional Streaming
        run_bidirectional_streaming_example(stub)
        
    finally:
        channel.close()
        print("\nConnection closed.")

if __name__ == '__main__':
    main()
```

## Async gRPC ile Yüksek Performans

AsyncIO ile non-blocking gRPC implementasyonu:

```python
# async_server.py
import asyncio
import grpc
from grpc import aio
import user_pb2
import user_pb2_grpc

class AsyncUserServiceServicer(user_pb2_grpc.UserServiceServicer):
    """Async UserService implementasyonu"""
    
    async def GetUser(self, request, context):
        """Async kullanıcı getirme"""
        # Async veritabanı sorgusu (örnek)
        await asyncio.sleep(0.1)  # Simulated DB query
        
        user = user_pb2.User(
            id=request.user_id,
            username=f"user_{request.user_id}",
            email=f"user{request.user_id}@example.com",
            is_active=True,
            roles=['user']
        )
        
        return user_pb2.GetUserResponse(
            user=user,
            message="User retrieved"
        )
    
    async def ListUsers(self, request, context):
        """Async server streaming"""
        for i in range(10):
            # Async işlem
            await asyncio.sleep(0.1)
            
            user = user_pb2.User(
                id=i,
                username=f"user_{i}",
                email=f"user{i}@example.com",
                is_active=True,
                roles=['user']
            )
            
            yield user

async def serve():
    """Async gRPC server"""
    server = aio.server()
    
    user_pb2_grpc.add_UserServiceServicer_to_server(
        AsyncUserServiceServicer(), server
    )
    
    listen_addr = '[::]:50051'
    server.add_insecure_port(listen_addr)
    
    print(f"Async gRPC Server starting on {listen_addr}")
    await server.start()
    
    try:
        await server.wait_for_termination()
    except KeyboardInterrupt:
        print("\nShutting down...")
        await server.stop(0)

if __name__ == '__main__':
    asyncio.run(serve())
```

Async client:

```python
# async_client.py
import asyncio
import grpc
from grpc import aio
import user_pb2
import user_pb2_grpc

async def run_async_calls():
    """Async gRPC çağrıları"""
    async with aio.insecure_channel('localhost:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        
        # Paralel unary çağrılar
        tasks = [
            stub.GetUser(user_pb2.GetUserRequest(user_id=i))
            for i in range(5)
        ]
        
        responses = await asyncio.gather(*tasks)
        
        print("Received users:")
        for response in responses:
            print(f"  - {response.user.username}")
        
        # Server streaming
        print("\nStreaming users:")
        request = user_pb2.ListUsersRequest(page=1, page_size=5)
        
        async for user in stub.ListUsers(request):
            print(f"  - {user.username}")

if __name__ == '__main__':
    asyncio.run(run_async_calls())
```

## Interceptors: Middleware Benzeri İşlemler

gRPC interceptor'ları ile logging, authentication, rate limiting gibi cross-cutting concerns implement edebilirsiniz:

```python
# interceptors.py
import grpc
import time
import logging
from grpc_interceptor import ServerInterceptor

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LoggingInterceptor(ServerInterceptor):
    """Request/response loglama"""
    
    def intercept(self, method, request, context, method_name):
        """Her RPC çağrısında çalışır"""
        start_time = time.time()
        
        logger.info(f"RPC Started: {method_name}")
        logger.info(f"Request: {request}")
        
        try:
            # Gerçek RPC çağrısı
            response = method(request, context)
            
            elapsed = time.time() - start_time
            logger.info(f"RPC Completed: {method_name} ({elapsed:.3f}s)")
            logger.info(f"Response: {response}")
            
            return response
        except Exception as e:
            elapsed = time.time() - start_time
            logger.error(f"RPC Failed: {method_name} ({elapsed:.3f}s) - {str(e)}")
            raise

class AuthInterceptor(ServerInterceptor):
    """Authentication kontrolü"""
    
    def intercept(self, method, request, context, method_name):
        """Token kontrolü yap"""
        # Metadata'dan token al
        metadata = dict(context.invocation_metadata())
        token = metadata.get('authorization', '')
        
        # Token validation (örnek)
        if not self.validate_token(token):
            context.abort(
                grpc.StatusCode.UNAUTHENTICATED,
                'Invalid or missing token'
            )
        
        # Token geçerli, devam et
        return method(request, context)
    
    def validate_token(self, token):
        """Token doğrulama"""
        # Gerçek implementasyonda JWT validation yapılır
        return token.startswith('Bearer ')

class RateLimitInterceptor(ServerInterceptor):
    """Rate limiting"""
    
    def __init__(self, max_requests_per_minute=60):
        self.max_requests = max_requests_per_minute
        self.requests = {}  # IP -> (count, window_start)
    
    def intercept(self, method, request, context, method_name):
        """Rate limit kontrolü"""
        # Client IP al
        peer = context.peer()
        
        current_time = time.time()
        
        if peer not in self.requests:
            self.requests[peer] = (1, current_time)
        else:
            count, window_start = self.requests[peer]
            
            # 1 dakikalık pencere kontrolü
            if current_time - window_start > 60:
                # Yeni pencere başlat
                self.requests[peer] = (1, current_time)
            else:
                # Aynı penceredeyiz
                if count >= self.max_requests:
                    context.abort(
                        grpc.StatusCode.RESOURCE_EXHAUSTED,
                        'Rate limit exceeded'
                    )
                
                self.requests[peer] = (count + 1, window_start)
        
        return method(request, context)

# Server'a interceptor ekleme
def serve_with_interceptors():
    """Interceptor'lı server"""
    from grpc_interceptor.server import ServerInterceptor as GRPCInterceptor
    
    interceptors = [
        LoggingInterceptor(),
        AuthInterceptor(),
        RateLimitInterceptor(max_requests_per_minute=100)
    ]
    
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
        interceptors=interceptors
    )
    
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(), server
    )
    
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()
```

## Error Handling ve Best Practices

### Proper Error Handling

```python
def GetUser(self, request, context):
    """Doğru error handling"""
    try:
        user_id = request.user_id
        
        # Validasyon
        if user_id <= 0:
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            context.set_details('User ID must be positive')
            return user_pb2.GetUserResponse()
        
        # Kullanıcıyı bul
        user = database.get_user(user_id)
        
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f'User {user_id} not found')
            return user_pb2.GetUserResponse()
        
        # Permission kontrolü
        if not has_permission(context, user):
            context.set_code(grpc.StatusCode.PERMISSION_DENIED)
            context.set_details('Insufficient permissions')
            return user_pb2.GetUserResponse()
        
        return user_pb2.GetUserResponse(user=user, message="Success")
        
    except DatabaseError as e:
        context.set_code(grpc.StatusCode.INTERNAL)
        context.set_details(f'Database error: {str(e)}')
        return user_pb2.GetUserResponse()
    
    except Exception as e:
        logger.exception("Unexpected error")
        context.set_code(grpc.StatusCode.UNKNOWN)
        context.set_details('Internal server error')
        return user_pb2.GetUserResponse()
```

### gRPC Status Codes

```python
# Yaygın kullanılan status code'lar
COMMON_STATUS_CODES = {
    grpc.StatusCode.OK: "Success",
    grpc.StatusCode.CANCELLED: "Request cancelled",
    grpc.StatusCode.INVALID_ARGUMENT: "Invalid argument provided",
    grpc.StatusCode.DEADLINE_EXCEEDED: "Deadline exceeded",
    grpc.StatusCode.NOT_FOUND: "Resource not found",
    grpc.StatusCode.ALREADY_EXISTS: "Resource already exists",
    grpc.StatusCode.PERMISSION_DENIED: "Permission denied",
    grpc.StatusCode.RESOURCE_EXHAUSTED: "Resource exhausted (rate limit)",
    grpc.StatusCode.FAILED_PRECONDITION: "Failed precondition",
    grpc.StatusCode.UNAUTHENTICATED: "Authentication required",
    grpc.StatusCode.INTERNAL: "Internal server error",
    grpc.StatusCode.UNAVAILABLE: "Service unavailable",
}
```

## Load Balancing ve Service Discovery

### Client-Side Load Balancing

```python
# load_balanced_client.py
import grpc

def create_load_balanced_channel():
    """Load balanced gRPC channel"""
    # DNS resolution ile load balancing
    # kubernetes gibi ortamlarda DNS round-robin
    options = [
        ('grpc.lb_policy_name', 'round_robin'),
        ('grpc.enable_retries', 1),
        ('grpc.keepalive_time_ms', 10000),
    ]
    
    # Service discovery DNS
    channel = grpc.insecure_channel(
        'user-service:50051',
        options=options
    )
    
    return channel

# Retry configuration
def create_channel_with_retry():
    """Retry politikası ile channel"""
    service_config = {
        'methodConfig': [{
            'name': [{'service': 'user.UserService'}],
            'retryPolicy': {
                'maxAttempts': 3,
                'initialBackoff': '0.1s',
                'maxBackoff': '1s',
                'backoffMultiplier': 2,
                'retryableStatusCodes': ['UNAVAILABLE', 'DEADLINE_EXCEEDED']
            }
        }]
    }
    
    channel = grpc.insecure_channel(
        'localhost:50051',
        options=[('grpc.service_config', json.dumps(service_config))]
    )
    
    return channel
```

## Testing gRPC Services

```python
# test_user_service.py
import pytest
import grpc
from grpc_testing import server_from_dictionary, strict_real_time
import user_pb2
import user_pb2_grpc
from server import UserServiceServicer

@pytest.fixture
def grpc_server():
    """Test gRPC server fixture"""
    servicer = UserServiceServicer()
    descriptors_to_services = {
        user_pb2.DESCRIPTOR.services_by_name['UserService']: servicer
    }
    
    return server_from_dictionary(
        descriptors_to_services,
        strict_real_time()
    )

def test_create_user(grpc_server):
    """Kullanıcı oluşturma testi"""
    method = grpc_server.invoke_unary_unary(
        user_pb2.DESCRIPTOR.services_by_name['UserService'].methods_by_name['CreateUser'],
        invocation_metadata={},
        request=user_pb2.CreateUserRequest(
            username="testuser",
            email="test@example.com",
            password="pass123"
        ),
        timeout=1
    )
    
    response, metadata, code, details = method.termination()
    
    assert code == grpc.StatusCode.OK
    assert response.success is True
    assert response.user.username == "testuser"

def test_get_user_not_found(grpc_server):
    """Kullanıcı bulunamadı testi"""
    method = grpc_server.invoke_unary_unary(
        user_pb2.DESCRIPTOR.services_by_name['UserService'].methods_by_name['GetUser'],
        invocation_metadata={},
        request=user_pb2.GetUserRequest(user_id=999),
        timeout=1
    )
    
    response, metadata, code, details = method.termination()
    
    assert code == grpc.StatusCode.NOT_FOUND
```

## Production Best Practices

### 1. TLS/SSL Güvenliği

```python
# secure_server.py
def serve_secure():
    """TLS ile güvenli server"""
    # SSL sertifikaları oku
    with open('server.key', 'rb') as f:
        private_key = f.read()
    
    with open('server.crt', 'rb') as f:
        certificate_chain = f.read()
    
    # Server credentials oluştur
    server_credentials = grpc.ssl_server_credentials(
        ((private_key, certificate_chain),)
    )
    
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(), server
    )
    
    # Güvenli port ekle
    server.add_secure_port('[::]:50051', server_credentials)
    server.start()
    server.wait_for_termination()
```

### 2. Health Check

```protobuf
// health.proto
syntax = "proto3";

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
  }
  ServingStatus status = 1;
}
```

### 3. Graceful Shutdown

```python
def serve_with_graceful_shutdown():
    """Graceful shutdown destekli server"""
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(), server
    )
    
    server.add_insecure_port('[::]:50051')
    server.start()
    
    print("Server started. Press Ctrl+C to stop.")
    
    def handle_sigterm(*_):
        print("Received shutdown signal")
        all_rpcs_done_event = server.stop(30)  # 30 saniye grace period
        all_rpcs_done_event.wait(30)
        print("Server stopped gracefully")
    
    import signal
    signal.signal(signal.SIGTERM, handle_sigterm)
    signal.signal(signal.SIGINT, handle_sigterm)
    
    server.wait_for_termination()
```

## Sonuç

gRPC, modern mikroservis mimarilerinde yüksek performans ve güçlü type-safety gerektiren senaryolar için mükemmel bir seçimdir. Protocol Buffers ile tanımlanan contract'lar sayesinde client ve server arasında güçlü bir tip güvenliği sağlanır, otomatik kod üretimi geliştirme sürecini hızlandırır.

Bu yazıda ele aldığımız konular:
- gRPC'nin temel kavramları ve REST ile karşılaştırması
- Protocol Buffers ile service tanımlama
- Python ile gRPC server ve client implementasyonu
- Unary, streaming (server/client/bidirectional) RPC tipleri
- Async gRPC ile yüksek performanslı uygulamalar
- Interceptor'lar ile middleware benzeri işlemler
- Error handling ve status code'lar
- Load balancing ve service discovery
- Testing stratejileri
- Production best practices (TLS, health check, graceful shutdown)

gRPC özellikle mikroservisler arası iletişim, IoT, mobil backend ve real-time uygulamalar için ideal bir seçimdir.

**Kaynaklar:**
- [gRPC Documentation](https://grpc.io/docs/)
- [Protocol Buffers Guide](https://protobuf.dev/)
- [gRPC Python Examples](https://github.com/grpc/grpc/tree/master/examples/python)

---
layout: default
title: "Подключение MongoDB к сервису"
permalink: /1. Погружение в основы/Подключение MongoDB к сервису/
---

*Также необходимо для одного из заданий. MongoDB - это NoSQL база данных. Не паникуй, если не знаешь, что это такое, скоро ты все поймешь.*
*NoSQL - это концепция баз данных "отличная от SQL", здесь каждая база данных имеет свои особенности. Например в Mongo все хранится в bson - это json в бинарном формате*

*MongoDB не так часто встречается на собеседованиях, но понимание основных ее принципов и работа с ней - это твой существенный козырь в рукаве*

## Установка драйвера MongoDB для Go
---
Первым делом установим официальный драйвер:

```bash
go get go.mongodb.org/mongo-driver/mongo
go get go.mongodb.org/mongo-driver/mongo/options
```

## Базовая настройка подключения
---
### Создаем модуль для работы с базой данных

```go
package database

import (
	"context"
	"fmt"
	"log"
	"time"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"go.mongodb.org/mongo-driver/mongo/readpref"
)

type MongoDB struct {
	Client   *mongo.Client
	Database *mongo.Database
}

func NewMongoDB(connectionString, dbName string) (*MongoDB, error) {
	// Создаем контекст с таймаутом для подключения
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Настраиваем опции подключения
	clientOptions := options.Client().
		ApplyURI(connectionString).
		SetMaxPoolSize(50).                    // Максимальное количество соединений в пуле
		SetMinPoolSize(10).                    // Минимальное количество соединений
		SetMaxConnIdleTime(5 * time.Minute).   // Максимальное время простаивания соединения
		SetServerSelectionTimeout(5 * time.Second)

	// Подключаемся к MongoDB
	client, err := mongo.Connect(ctx, clientOptions)
	if err != nil {
		return nil, fmt.Errorf("ошибка подключения к MongoDB: %v", err)
	}

	// Проверяем подключение
	err = client.Ping(ctx, readpref.Primary())
	if err != nil {
		return nil, fmt.Errorf("ошибка ping MongoDB: %v", err)
	}

	log.Println("Успешное подключение к MongoDB!")

	return &MongoDB{
		Client:   client,
		Database: client.Database(dbName),
	}, nil
}

func (m *MongoDB) Close() error {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	return m.Client.Disconnect(ctx)
}
```

## Интеграция MongoDB в Order Service
---
### Обновляем структуру сервера

```go
package main

import (
	"context"
	"log"
	"net"
	"time"

	"your-project/order-service/database"
	"your-project/order-service/proto/order"
	"your-project/user-service/proto/user"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type OrderServer struct {
	order.UnimplementedOrderServiceServer
	userClient    user.UserServiceClient
	db            *database.MongoDB
	orderCollection *mongo.Collection
}

func NewOrderServer(userServiceURL, mongoURL string) (*OrderServer, error) {
	// Подключаемся к MongoDB
	db, err := database.NewMongoDB(mongoURL, "orderservice")
	if err != nil {
		return nil, err
	}

	// Подключаемся к User Service
	conn, err := grpc.Dial(userServiceURL, 
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithTimeout(5*time.Second))
	
	if err != nil {
		db.Close()
		return nil, err
	}

	server := &OrderServer{
		userClient:      user.NewUserServiceClient(conn),
		db:              db,
		orderCollection: db.Database.Collection("orders"),
	}

	// Создаем индексы
	go server.createIndexes()

	return server, nil
}

func (s *OrderServer) createIndexes() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Создаем индекс для поиска по user_id
	indexModel := mongo.IndexModel{
		Keys: bson.D{{Key: "user_id", Value: 1}},
	}
	_, err := s.orderCollection.Indexes().CreateOne(ctx, indexModel)
	if err != nil {
		log.Printf("Ошибка создания индекса: %v", err)
	} else {
		log.Println("Индексы MongoDB созданы успешно")
	}
}
```

## Определяем структуры данных для MongoDB
---
### Модели данных

```go
package models

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type Order struct {
	ID          primitive.ObjectID `bson:"_id,omitempty" json:"id"`
	OrderID     string             `bson:"order_id" json:"order_id"`
	UserID      string             `bson:"user_id" json:"user_id"`
	UserName    string             `bson:"user_name" json:"user_name"`
	Items       []OrderItem        `bson:"items" json:"items"`
	TotalAmount float64            `bson:"total_amount" json:"total_amount"`
	Status      string             `bson:"status" json:"status"`
	CreatedAt   time.Time          `bson:"created_at" json:"created_at"`
	UpdatedAt   time.Time          `bson:"updated_at" json:"updated_at"`
}

type OrderItem struct {
	ProductID string  `bson:"product_id" json:"product_id"`
	Quantity  int32   `bson:"quantity" json:"quantity"`
	Price     float64 `bson:"price" json:"price"`
}

// ToProto преобразует Order в protobuf сообщение
func (o *Order) ToProto() *order.CreateOrderResponse {
	return &order.CreateOrderResponse{
		OrderId:     o.OrderID,
		Status:      o.Status,
		UserName:    o.UserName,
		TotalAmount: o.TotalAmount,
	}
}
```

## Реализуем CRUD операции с MongoDB

---

### Репозиторий для работы с заказами

```go
package repository

import (
	"context"
	"time"

	"your-project/order-service/models"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

type OrderRepository struct {
	collection *mongo.Collection
}

func NewOrderRepository(collection *mongo.Collection) *OrderRepository {
	return &OrderRepository{
		collection: collection,
	}
}

func (r *OrderRepository) CreateOrder(ctx context.Context, order *models.Order) error {
	order.CreatedAt = time.Now()
	order.UpdatedAt = time.Now()

	result, err := r.collection.InsertOne(ctx, order)
	if err != nil {
		return err
	}

	// Устанавливаем сгенерированный ID
	if oid, ok := result.InsertedID.(primitive.ObjectID); ok {
		order.ID = oid
	}

	return nil
}

func (r *OrderRepository) GetOrder(ctx context.Context, orderID string) (*models.Order, error) {
	var order models.Order
	
	err := r.collection.FindOne(ctx, bson.M{"order_id": orderID}).Decode(&order)
	if err != nil {
		if err == mongo.ErrNoDocuments {
			return nil, nil // Заказ не найден
		}
		return nil, err
	}

	return &order, nil
}

func (r *OrderRepository) GetUserOrders(ctx context.Context, userID string, limit, offset int64) ([]*models.Order, error) {
	var orders []*models.Order

	findOptions := options.Find()
	findOptions.SetSort(bson.D{{Key: "created_at", Value: -1}}) // Сортировка по дате создания (новые first)
	findOptions.SetLimit(limit)
	findOptions.SetSkip(offset)

	cursor, err := r.collection.Find(ctx, bson.M{"user_id": userID}, findOptions)
	if err != nil {
		return nil, err
	}
	defer cursor.Close(ctx)

	for cursor.Next(ctx) {
		var order models.Order
		if err := cursor.Decode(&order); err != nil {
			return nil, err
		}
		orders = append(orders, &order)
	}

	if err := cursor.Err(); err != nil {
		return nil, err
	}

	return orders, nil
}

func (r *OrderRepository) UpdateOrderStatus(ctx context.Context, orderID, status string) error {
	_, err := r.collection.UpdateOne(
		ctx,
		bson.M{"order_id": orderID},
		bson.M{
			"$set": bson.M{
				"status":     status,
				"updated_at": time.Now(),
			},
		},
	)
	return err
}

func (r *OrderRepository) DeleteOrder(ctx context.Context, orderID string) error {
	_, err := r.collection.DeleteOne(ctx, bson.M{"order_id": orderID})
	return err
}
```

## Обновляем gRPC методы с использованием MongoDB
---
### Переписываем метод CreateOrder

```go
func (s *OrderServer) CreateOrder(ctx context.Context, req *order.CreateOrderRequest) (*order.CreateOrderResponse, error) {
	log.Printf("Создание заказа %s для пользователя %s", req.OrderId, req.UserId)

	// Сначала проверяем, не существует ли уже заказ с таким ID
	existingOrder, err := s.orderRepository.GetOrder(ctx, req.OrderId)
	if err != nil {
		return &order.CreateOrderResponse{
			OrderId:      req.OrderId,
			Status:       "ERROR",
			ErrorMessage: "Ошибка проверки заказа: " + err.Error(),
		}, nil
	}

	if existingOrder != nil {
		return &order.CreateOrderResponse{
			OrderId:      req.OrderId,
			Status:       "ERROR", 
			ErrorMessage: "Заказ с таким ID уже существует",
		}, nil
	}

	// Валидируем пользователя через User Service
	userCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	validateReq := &user.ValidateUserRequest{UserId: req.UserId}
	validateResp, err := s.userClient.ValidateUser(userCtx, validateReq)
	
	if err != nil {
		return &order.CreateOrderResponse{
			OrderId:      req.OrderId,
			Status:       "ERROR",
			ErrorMessage: "Ошибка валидации пользователя: " + err.Error(),
		}, nil
	}
	
	if !validateResp.Valid {
		return &order.CreateOrderResponse{
			OrderId:      req.OrderId,
			Status:       "ERROR", 
			ErrorMessage: validateResp.ErrorMessage,
		}, nil
	}

	// Рассчитываем общую сумму
	total := 0.0
	for _, item := range req.Items {
		total += item.Price * float64(item.Quantity)
	}

	// Создаем объект заказа
	newOrder := &models.Order{
		OrderID:     req.OrderId,
		UserID:      req.UserId,
		UserName:    validateResp.UserName,
		Items:       convertProtoItemsToModels(req.Items),
		TotalAmount: total,
		Status:      "CREATED",
	}

	// Сохраняем в MongoDB
	err = s.orderRepository.CreateOrder(ctx, newOrder)
	if err != nil {
		return &order.CreateOrderResponse{
			OrderId:      req.OrderId,
			Status:       "ERROR",
			ErrorMessage: "Ошибка сохранения заказа: " + err.Error(),
		}, nil
	}

	log.Printf("Заказ %s успешно создан в MongoDB", req.OrderId)
	return newOrder.ToProto(), nil
}

func convertProtoItemsToModels(protoItems []*order.OrderItem) []models.OrderItem {
	var items []models.OrderItem
	for _, protoItem := range protoItems {
		items = append(items, models.OrderItem{
			ProductID: protoItem.ProductId,
			Quantity:  protoItem.Quantity,
			Price:     protoItem.Price,
		})
	}
	return items
}
```

## Добавляем новые методы для работы с заказами
---
### Расширяем proto файл

```protobuf
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse) {}
  rpc GetOrder (GetOrderRequest) returns (GetOrderResponse) {}
  rpc GetUserOrders (GetUserOrdersRequest) returns (GetUserOrdersResponse) {}
  rpc UpdateOrderStatus (UpdateOrderStatusRequest) returns (UpdateOrderStatusResponse) {}
}

message GetOrderRequest {
  string order_id = 1;
}

message GetOrderResponse {
  Order order = 1;
  string error_message = 2;
}

message GetUserOrdersRequest {
  string user_id = 1;
  int32 limit = 2;
  int32 offset = 3;
}

message GetUserOrdersResponse {
  repeated Order orders = 1;
  int32 total_count = 2;
  string error_message = 3;
}

message UpdateOrderStatusRequest {
  string order_id = 1;
  string status = 2;
}

message UpdateOrderStatusResponse {
  bool success = 1;
  string error_message = 2;
}

// Расширяем сообщение Order для полного представления
message Order {
  string order_id = 1;
  string user_id = 2;
  string user_name = 3;
  repeated OrderItem items = 4;
  double total_amount = 5;
  string status = 6;
  string created_at = 7;
  string updated_at = 8;
}
```

### Реализуем новые методы

```go
func (s *OrderServer) GetOrder(ctx context.Context, req *order.GetOrderRequest) (*order.GetOrderResponse, error) {
	foundOrder, err := s.orderRepository.GetOrder(ctx, req.OrderId)
	if err != nil {
		return &order.GetOrderResponse{
			ErrorMessage: "Ошибка поиска заказа: " + err.Error(),
		}, nil
	}

	if foundOrder == nil {
		return &order.GetOrderResponse{
			ErrorMessage: "Заказ не найден",
		}, nil
	}

	return &order.GetOrderResponse{
		Order: convertModelToProtoOrder(foundOrder),
	}, nil
}

func (s *OrderServer) GetUserOrders(ctx context.Context, req *order.GetUserOrdersRequest) (*order.GetUserOrdersResponse, error) {
	limit := int64(10)
	if req.Limit > 0 {
		limit = int64(req.Limit)
	}

	offset := int64(0)
	if req.Offset > 0 {
		offset = int64(req.Offset)
	}

	orders, err := s.orderRepository.GetUserOrders(ctx, req.UserId, limit, offset)
	if err != nil {
		return &order.GetUserOrdersResponse{
			ErrorMessage: "Ошибка получения заказов: " + err.Error(),
		}, nil
	}

	// Конвертируем модели в proto сообщения
	var protoOrders []*order.Order
	for _, ord := range orders {
		protoOrders = append(protoOrders, convertModelToProtoOrder(ord))
	}

	return &order.GetUserOrdersResponse{
		Orders:     protoOrders,
		TotalCount: int32(len(protoOrders)),
	}, nil
}

func (s *OrderServer) UpdateOrderStatus(ctx context.Context, req *order.UpdateOrderStatusRequest) (*order.UpdateOrderStatusResponse, error) {
	err := s.orderRepository.UpdateOrderStatus(ctx, req.OrderId, req.Status)
	if err != nil {
		return &order.UpdateOrderStatusResponse{
			Success:      false,
			ErrorMessage: "Ошибка обновления статуса: " + err.Error(),
		}, nil
	}

	return &order.UpdateOrderStatusResponse{
		Success: true,
	}, nil
}

func convertModelToProtoOrder(model *models.Order) *order.Order {
	return &order.Order{
		OrderId:     model.OrderID,
		UserId:      model.UserID,
		UserName:    model.UserName,
		Items:       convertModelItemsToProto(model.Items),
		TotalAmount: model.TotalAmount,
		Status:      model.Status,
		CreatedAt:   model.CreatedAt.Format(time.RFC3339),
		UpdatedAt:   model.UpdatedAt.Format(time.RFC3339),
	}
}
```

## Конфигурация и переменные окружения
---
### Файл конфигурации

```go
package config

import (
	"os"
	"strconv"
)

type Config struct {
	Port          string
	UserServiceURL string
	MongoURL      string
	DatabaseName  string
}

func Load() *Config {
	return &Config{
		Port:          getEnv("PORT", "50052"),
		UserServiceURL: getEnv("USER_SERVICE_URL", "localhost:50051"),
		MongoURL:      getEnv("MONGO_URL", "mongodb://localhost:27017"),
		DatabaseName:  getEnv("DATABASE_NAME", "orderservice"),
	}
}

func getEnv(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}
```

## Обновляем main.go
---
### Финальная версия main.go

```go
package main

import (
	"log"
	"net"

	"your-project/order-service/config"
	"your-project/order-service/database"
	"your-project/order-service/proto/order"
	"your-project/order-service/repository"

	"google.golang.org/grpc"
)

func main() {
	// Загружаем конфигурацию
	cfg := config.Load()

	// Создаем сервер Order Service
	server, err := NewOrderServer(cfg.UserServiceURL, cfg.MongoURL)
	if err != nil {
		log.Fatalf("Не удалось создать сервер: %v", err)
	}
	defer server.db.Close()

	// Инициализируем репозиторий
	server.orderRepository = repository.NewOrderRepository(server.orderCollection)

	// Запускаем gRPC сервер
	lis, err := net.Listen("tcp", ":"+cfg.Port)
	if err != nil {
		log.Fatalf("Ошибка запуска сервера: %v", err)
	}

	grpcServer := grpc.NewServer()
	order.RegisterOrderServiceServer(grpcServer, server)

	log.Printf("Order Service запущен на порту %s", cfg.Port)
	
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("Ошибка сервера: %v", err)
	}
}
```

## Docker Compose для локальной разработки
---
### docker-compose.yml

```yaml
version: '3.8'
services:
  mongodb:
    image: mongo:6.0
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - mongodb_data:/data/db

  user-service:
    build: ./user-service
    ports:
      - "50051:50051"
    environment:
      - PORT=50051

  order-service:
    build: ./order-service  
    ports:
      - "50052:50052"
    environment:
      - PORT=50052
      - USER_SERVICE_URL=user-service:50051
      - MONGO_URL=mongodb://admin:password@mongodb:27017
      - DATABASE_NAME=orderservice
    depends_on:
      - mongodb
      - user-service

volumes:
  mongodb_data:
```

---

**[← Назад к Backend](1.%20Погружение%20в%20основы/Backend/)**
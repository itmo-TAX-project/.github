# Taxi: Tax
## Бизнес Процессы
##### 1) Клиенты:
- зарегистрироваться
- войти / выйти из аккаунта
- пополнить счёт (всё что попало на счёт, остаётся там, возврату не подлежит, при возврате средств за поездку средства приходят на этот счёт)
- запросить поездку
- отменить поездку
- оставить отзыв (простенький рейтинговый сервис, просто поставить количество звёзд водителю)
- запросить возврат средств на счёт
- оставить чаевые
##### 2) Водители:
- зарегистрироваться
- войти / выйти из аккаунта
- сменить автомобиль (базовый сегмент, средний класс, премиум сегмент)
- искать клиентов
- отмена поиска клиентов
- получить маршрут до клиента
- отменить поездку
- успешно завершить поездку
- оставить отзыв для пассажира (тот же рейтинговый сервис, просто поставить количество звёзд пассажиру)
##### 3) Администраторы:
- войти / выйти из аккаунта
- создать нового администратора
- разрешить ситуацию для возврата средств на счёт клиента
- забанить/разбанить водителя
- забанить/разбанить клиента
## Сервисы
### 1. HTTP Gateway
#### Middleware

1. JWT Validation:
* Проверяет валидность токена.
* Извлекает роль пользователя.
2. Ban Check:
* Gateway обращается к сервису AccountService по gRPC, который хранит флаг banned, обновляемый через Kafka (admin.ban_changed).
* Запрещает операции заблокированным пользователям.

#### REST endpoints
- POST /auth/register (role: passenger/driver/admin) - создает аккаунт в AccountService, возвращает JWT токен. После записи в AccountService Gateway вызывает gRPC к соответствующему мастер-сервису.
- POST /auth/login
- POST /wallet/topup/{amount} - пополнение денег
- POST /rides - инициализация поиска поездки пассажиром rideId
- POST /rides/cancel/{rideId} - отмена поездки пассажиром
- POST /drivers/{id}/switch-car - смена сегмента авто
- POST /drivers/search-ride - поиск пассажиров водителем
- POST /drivers/cancel-searching - отмена поиска пассажиров водителем
- POST /rides/{rideId}/confirm - принятие заказа для водителя
- POST /rides/{rideId}/start- начало поездки для водителя
- POST /rides/{rideId}/finish- выполнение поездки для водителя
- POST /ratings - поставить звезду
- POST /rides/refunds/request/{rideId} - запрос возврата
- POST /rides/tip/{rideId} - чаевые 
- GET /track/{rideId} - агрегирует данные
- POST /admin/add - создание нового админа
- POST /rides/refund/resolve/{rideId} - разрешить ситуацию для возврата средств на счет клиента
- POST /admin/ban/{id} - бан пользователя
- POST /admin/unban/{id} - разбан пользователя

#### WebSocket'ы
- ride.driver_assigned - для получения passenger'у информации о найденном водителе для поездки
- ride.passenger_assigned - для предложение водителю заказа от какого-то пассажира (который он будет подтверждать через http эндпоинт)
- ride.status_changed - изменения статуса поездки

### 2. Account Service - мастер система для аккаунтов
#### Сущности
- Account { account_id, phone, password_hash, role (passenger|driver|admin), banned, created_at }
#### gRPC
```
service AccountService {
  rpc CreateAccount(CreateAccountRequest) returns (CreateAccountResponse);
  rpc GetAccountById(GetAccountByIdRequest) returns (GetAccountByIdResponse);
  rpc CheckBanStatus(CheckBanStatusRequest) returns (CheckBanStatusResponse);
  rpc SetBanStatus(SetBanStatusRequest) returns (SetBanStatusResponse);
}
```
#### Kafka
- Subscribes: admin.ban_changed
### 3. Taxi-Service — мастер-система для водителей и автомобилей
#### Сущности
- Driver { driver_id, account_id, name, license_number, current_vehicle_id, allowed_segments, rating }
- Vehicle { vehicle_id, driver_id, segment (basic/mid/premium), plate, model, capacity }
- Snapshot: DriverStatus { driver_id, location, availability (searching/idle/busy/offline), ts }
#### gRPC
```
service TaxiService{
  rpc GetDriver(GetDriverRequest) returns (GetDriverResponse);
  rpc ValidateDriverActive(ValidateDriverRequest) returns (ValidateDriverResponse);
}
```
ValidateDriverActive — проверка активен ли водитель в данный момент
#### Kafka
* Publishes: taxi_driver_created, taxi_driver_status_changed, taxi_vehicle_changed.
* Subscribes: taxi_driver_create, taxi_driver_status_change, taxi_vehicle_change, taxi_vehicle_create, account_deleted.
### 4. Passenger-Service — мастер-система для пассажиров
#### Сущности
- Passenger { passenger_id, account_id, name, phone, created_at }
- PassengerPreferences { payment_default, allowed_segments }
#### gRPC
```
service PassengerMaster {
  rpc CreatePassenger(CreatePassengerRequest) returns (CreatePassengerResponse);
  rpc GetPassenger(GetPassengerRequest) returns (GetPassengerResponse);
}
```
#### Kafka
Publishes: passenger.created.
Subscribes: account.created.
### 5. Admin-Service — мастер-система для администраторов
#### Сущности
- Admin { admin_id, account_id, role }
- BanRecord { subject_type (driver|passenger), subject_id, created_by_admin_id, reason, ts, expires_at }
- RefundRequest { request_id, ride_id, passenger_id, amount, status }
#### gRPC
```
service AdminMaster {
  rpc CreateAdmin(CreateAdminRequest) returns (CreateAdminResponse);
  rpc BanEntity(BanEntityRequest) returns (BanEntityResponse);
  rpc UnbanEntity(UnbanEntityRequest) returns (UnbanEntityResponse);
  rpc ApproveRefund(ApproveRefundRequest) returns (ApproveRefundResponse);
}
```
#### Kafka
Publishes: admin.ban_changed, admin.refund_approved.
### 6. Ride Service - управление поездками
Здесь водитель принимает заказ. (сказать на защите)
#### Сущности
- Ride { ride_id, passenger_id, pickup, dropoff, status, requested_at, assigned_driver_id, estimated_price, final_price, route_id }
#### gRPC
```
service RideService {
  rpc CreateRide(CreateRideRequest) returns (CreateRideResponse);
  rpc CancelRide(CancelRideRequest) returns (CancelRideResponse);
  rpc GetRide(GetRideRequest) returns (RideDto);
  rpc UpdateRideStatus(UpdateRideStatusRequest) returns (UpdateRideStatusResponse);
}
```
#### Kafka
Publishes: ride.requested, ride.cancelled, ride.started, ride.completed, ride.assigned (публикует в ответ на dispatch).
Subscribes: dispatch.driver_assigned.
### 7. Route Service - построение маршрутов и ETA
Расчёт маршрутов между точками, выдача полных путей для драйвера.
#### Сущности
- Route { route_id, path_polyline, distance_m, duration_s, segments }
#### gRPC
```
service RouteService {
  rpc CalculateRoute(CalculateRouteRequest) returns (CalculateRouteResponse);
  rpc GetRoute(GetRouteRequest) returns (GetRouteResponse);
}
```
#### Хранилище
Используем локальную картографическую библиотеку. (сказать на защите)
### 8. Dispatch Service - только Kafka (только реактивноe взаимодействие, сказать на защите)
Подбирает кандидатов водителей для перевозки пассажира
#### Логика
- Хранит локальный snapshot доступных водителей (обновляется из taxi.driver.status_changed).
- На ride.requested - фильтрует кандидатов по сегменту авто, расстоянию, рейтингу и публикует dispatch.driver_assigned(ride_id, driver_id, eta).
#### Валидации
Необходимо TaxiMaster.ValidateDriverActive(driver_id).
#### Kafka
Subscribes: ride.requested, taxi.driver.status_changed, ride.cancelled.
Publishes: dispatch.driver_assigned, dispatch.assignment_failed.
### 9. Finance Service - кошельки и расчеты
Управляет кошельками пассажиров (ручка top-up в gateway), проводит списания за поездку, возврат средств на счет (по админ-решению), начисление чаевых водителю.
#### Сущности
- Wallet { wallet_id, owner_id (passenger|driver), balance_cents }
- Transaction { id, wallet_id, type (topup|charge|tip|refund|payout), amount, status, created_at }
#### gRPC
```
service FinanceService {
  rpc TopUp(TopUpRequest) returns (TopUpResponse);
  rpc ChargeForRide(ChargeRequest) returns (ChargeForRideResponse);
  rpc RequestRefund(RefundRequest) returns (RequestRefundResponse);
  rpc Tip(TipRequest) returns (TipResponse);
  rpc PayoutToDriver(PayoutRequest) returns (PayoutToDriverResponse);
}
```
PayoutToDriver вызывает TaxiMaster.ValidateDriverActive(driver_id) через gRPC.
#### Kafka
Subscribes: ride.completed -> ChargeForRide; admin.refund_approved -> perform refund; tip.added.
#### Логика
Пополнение на счет - средства остаются на счёте, не возвращаются на внешние карты. При возврате за поездку средства приходят на этот же внутренний счет. (сказать на защите)
### 10. Rating Service - рейтинги водителя и пассаж
Управляет рейтингами и отзывами.
#### Сущности
Rating { subject_type (driver|passenger), subject_id, rater_id, stars, comment, created_at }
RatingAggregate { subject_id, avg, count }
#### gRPC
```
service RatingService {
  rpc AddRating(AddRatingRequest) returns (AddRatingResponse);
  rpc GetRating(GetRatingRequest) returns (GetRatingResponse);
}
```
#### Kafka
Subscribes: rating.added.

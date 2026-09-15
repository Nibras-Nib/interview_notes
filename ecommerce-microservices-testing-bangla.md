# E-commerce Microservices টেস্টিং ডকুমেন্টেশন (একদম শুরু থেকে বাংলায়)

এই লেসনে আমরা একটা সম্পূর্ণ **Microservices-ভিত্তিক E-commerce সিস্টেম** নিয়ে আলোচনা করব — এতে একাধিক ছোট ছোট স্বাধীন সার্ভিস (microservice) একসাথে মিলে একটা সম্পূর্ণ অনলাইন শপিং সিস্টেম তৈরি করে। আমরা প্রথমে পুরো আর্কিটেকচার (architecture) বুঝব, তারপর প্রতিটা সার্ভিস আলাদাভাবে দেখব, এবং সবশেষে হাতেকলমে `curl` কমান্ড দিয়ে পুরো সিস্টেমটা টেস্ট করব।

---

## ১. Microservices Architecture কী?

**Microservices** হলো একটা সফটওয়্যার ডিজাইন পদ্ধতি, যেখানে একটা বড় অ্যাপ্লিকেশনকে একটা মাত্র বিশাল প্রোগ্রাম (monolith) হিসেবে না বানিয়ে, বরং অনেকগুলো **ছোট ছোট, স্বাধীন সার্ভিস**-এ ভাগ করে বানানো হয়। প্রতিটা সার্ভিস নিজের একটা নির্দিষ্ট কাজের (যেমন ইউজার ম্যানেজমেন্ট, প্রোডাক্ট ম্যানেজমেন্ট) দায়িত্বে থাকে, এবং প্রয়োজনে একে অপরের সাথে যোগাযোগ করে।

**চিত্র ১** আমাদের এই সিস্টেমের পুরো আর্কিটেকচার দেখাচ্ছে, যেটা তিনটা স্তরে (layer) ভাগ করা:

### স্তর ১: API Gateway Layer
- **Client** (ব্যবহারকারী বা ফ্রন্টএন্ড অ্যাপ) প্রথমে যোগাযোগ করে **Nginx API Gateway**-এর সাথে
- **API Gateway কী?** এটা একটা "ট্রাফিক পুলিশ"-এর মতো কাজ করে — client-এর রিকোয়েস্ট আসলে কোন সার্ভিসের জন্য, সেটা বুঝে সঠিক সার্ভিসের কাছে পাঠিয়ে দেয়। Client-কে জানতেও হয় না কতগুলো আলাদা সার্ভিস আছে, বা কোনটা কোথায় চলছে — শুধু একটা ঠিকানায় (Nginx) রিকোয়েস্ট পাঠালেই চলে।
- Nginx বিভিন্ন URL path অনুযায়ী রিকোয়েস্ট আলাদা আলাদা সার্ভিসে রুট (route) করে দেয়:
  - `/api/v1/users` এবং `/api/v1/auth` → **User Service**-এ যায়
  - `/api/v1/products` → **Product Service**-এ যায়
  - `/api/v1/orders` → **Order Service**-এ যায়
  - `/api/v1/inventory` → **Inventory Service**-এ যায়

### স্তর ২: Microservices Layer
এখানে চারটা আলাদা **FastAPI**-ভিত্তিক সার্ভিস আছে:
- **User Service** — ব্যবহারকারী ব্যবস্থাপনা এবং authentication
- **Product Service** — প্রোডাক্ট তথ্য ব্যবস্থাপনা
- **Order Service** — অর্ডার প্রসেসিং, যেটা **অন্য সব সার্ভিসের সাথেই** যোগাযোগ করে (User, Product, Inventory)
- **Inventory Service** — স্টক/মজুদ ব্যবস্থাপনা

**FastAPI কী?** এটা Python-এর একটা আধুনিক, দ্রুতগতির ওয়েব ফ্রেমওয়ার্ক, যেটা দিয়ে সহজে এবং দ্রুত API বানানো যায়।

### স্তর ৩: Database Layer
প্রতিটা সার্ভিসের **নিজস্ব আলাদা ডেটাবেজ** আছে — এটাই microservices-এর একটা গুরুত্বপূর্ণ নীতি (প্রতিটা সার্ভিস তার নিজের ডেটার মালিক, অন্য সার্ভিস সরাসরি অন্যের ডেটাবেজে ঢুকতে পারে না):
- User Service → **PostgreSQL** (User Data)
- Product Service → **MongoDB** (Product Data)
- Order Service → **MongoDB** (Order Data)
- Inventory Service → **PostgreSQL** (Inventory Data)

**PostgreSQL বনাম MongoDB:** PostgreSQL হলো একটা **relational (SQL)** ডেটাবেজ, যেখানে ডেটা টেবিলের সারি-কলামে সাজানো থাকে এবং টেবিলগুলোর মধ্যে সম্পর্ক (relationship) তৈরি করা যায়। MongoDB হলো একটা **NoSQL (document-based)** ডেটাবেজ, যেখানে ডেটা JSON-এর মতো "document" আকারে সংরক্ষিত হয়, এবং এর গঠন (structure) তুলনামূলক নমনীয় (flexible)।

---

## ২. প্রতিটা Microservice-এর সাধারণ প্রজেক্ট গঠন

প্রতিটা সার্ভিসের ফোল্ডার গঠন প্রায় একই রকম প্যাটার্ন অনুসরণ করে:

```
service-name/
├── .env                    # Environment variables
├── Dockerfile              # Container কনফিগারেশন
├── docker-compose.yml      # Local development সেটআপ
├── requirements.txt        # Python dependency তালিকা
├── README.md               # সার্ভিসের ডকুমেন্টেশন
└── app/
    ├── main.py              # FastAPI অ্যাপ্লিকেশনের শুরুর পয়েন্ট
    ├── api/                 # API layer (routes, dependencies)
    ├── core/                # Core সেটিংস এবং utility
    ├── db/                  # ডেটাবেজ সংযোগ লজিক
    ├── models/              # ডেটা মডেল (Pydantic/ORM)
    └── services/            # অন্য সার্ভিসের সাথে যোগাযোগের কোড
```

এই গঠনটা একটা প্রমিত (standardized) টেমপ্লেট, যাতে প্রতিটা সার্ভিসের কোড একই রকম প্যাটার্নে সংগঠিত থাকে — এতে ভিন্ন সার্ভিসে কাজ করার সময়ও ডেভেলপারদের বুঝতে সুবিধা হয়, কারণ ফোল্ডার-গঠন পরিচিত থাকে।

---

## ৩. User Service — বিস্তারিত বিশ্লেষণ

**User Service** হলো একটা FastAPI-ভিত্তিক সার্ভিস, যেটা ব্যবহারকারী ব্যবস্থাপনা, **JWT টোকেন** দিয়ে authentication (প্রমাণীকরণ) এবং authorization (অনুমোদন) নিয়ন্ত্রণ করে। এটা ডেটাবেজ হিসেবে **PostgreSQL** ব্যবহার করে।

**JWT (JSON Web Token) কী?** এটা একটা সুরক্ষিত টোকেন (এক ধরনের encoded স্ট্রিং), যেটা ইউজার লগইন করার পর তাকে দেওয়া হয়। এরপর প্রতিটা রিকোয়েস্টে এই টোকেনটা পাঠালে সার্ভার বুঝতে পারে ইউজার আসলে কে, এবং সে লগইন করা আছে কিনা — বারবার username/password পাঠানোর দরকার হয় না।

**চিত্র ২** এই সার্ভিসের ভেতরের গঠন দেখাচ্ছে (Port 8003-এ চলে):
- **API Endpoints** — register, login, profile দেখা, address যোগ করা
- **Authentication** — JWT টোকেন তৈরি, পাসওয়ার্ড হ্যাশিং, টোকেন যাচাই
- **Business Logic** — রেজিস্ট্রেশন, প্রোফাইল/ঠিকানা ব্যবস্থাপনা, ডেটা যাচাই
- **Database Operations** — Async SQLAlchemy ORM, Connection Pooling, Transaction Management

### ডেটাবেজ মডেল

**Users টেবিল:**

```python
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    first_name = Column(String, nullable=False)
    last_name = Column(String, nullable=False)
    phone = Column(String, nullable=True)
    is_active = Column(Boolean, nullable=False, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    
    addresses = relationship("Address", back_populates="user", cascade="all, delete-orphan")
```

কী কী গুরুত্বপূর্ণ পয়েন্ট এখানে আছে:
- `email = Column(String, unique=True, ...)` → email অবশ্যই **ইউনিক (unique)** হতে হবে, মানে দুইজন ইউজার একই email দিয়ে রেজিস্ট্রেশন করতে পারবে না
- `hashed_password` → লক্ষ্য করো, এখানে সরাসরি পাসওয়ার্ড সেভ করা হয় না, বরং **hash** করা (একমুখী এনক্রিপশন করা) পাসওয়ার্ড সেভ হয় — এটা নিরাপত্তার জন্য অত্যন্ত গুরুত্বপূর্ণ, যাতে ডেটাবেজ চুরি হয়ে গেলেও আসল পাসওয়ার্ড কেউ দেখতে না পারে
- `relationship("Address", ..., cascade="all, delete-orphan")` → এটা বলে দেয় যে একজন User-এর একাধিক Address থাকতে পারে, এবং যদি User ডিলিট হয়, তাহলে তার সব Address-ও স্বয়ংক্রিয়ভাবে ডিলিট হয়ে যাবে (`cascade` মানে "চেইন রিঅ্যাকশন")

**Addresses টেবিল:**

```python
class Address(Base):
    __tablename__ = "addresses"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    line1 = Column(String, nullable=False)
    line2 = Column(String, nullable=True)
    city = Column(String, nullable=False)
    state = Column(String, nullable=False)
    postal_code = Column(String, nullable=False)
    country = Column(String, nullable=False)
    is_default = Column(Boolean, nullable=False, default=False)
```

- `user_id = Column(Integer, ForeignKey("users.id", ...))` → এটা একটা **Foreign Key**, যেটা এই Address-টা ঠিক কোন User-এর সাথে সম্পর্কিত সেটা বলে দেয়। এভাবেই দুটো টেবিলের মধ্যে সম্পর্ক (relationship) তৈরি হয়।
- `is_default` → একজন ইউজারের একাধিক ঠিকানার মধ্যে কোনটা ডিফল্ট (main) ঠিকানা, সেটা চিহ্নিত করা হয়

### API Routes (এন্ডপয়েন্ট)

**Authentication Routes:**
- `POST /api/v1/auth/register` → নতুন ইউজার রেজিস্টার করে (email, password যাচাই করে)
- `POST /api/v1/auth/login` → ইউজারকে যাচাই (authenticate) করে এবং access ও refresh টোকেন রিটার্ন করে
- `POST /api/v1/auth/refresh` → পুরনো refresh টোকেন দিয়ে নতুন access টোকেন নেওয়া যায় (বারবার লগইন করতে হয় না)

**User Management Routes:**
- `GET /api/v1/users/me` → নিজের প্রোফাইল এবং ঠিকানা দেখা
- `PUT /api/v1/users/me` → প্রোফাইল তথ্য আপডেট করা
- `PUT /api/v1/users/me/password` → পাসওয়ার্ড পরিবর্তন করা

**Address Management Endpoints:**
- `GET /api/v1/users/me/addresses` → সব ঠিকানার তালিকা দেখা
- `POST /api/v1/users/me/addresses` → নতুন ঠিকানা যোগ করা
- `GET /api/v1/users/me/addresses/{address_id}` → নির্দিষ্ট একটা ঠিকানা দেখা

**User Verification (Inter-service):**
- `GET /api/v1/users/{user_id}/verify` → এই endpoint-টা authentication লাগে না, কারণ এটা **অন্য সার্ভিস** (যেমন Order Service) ব্যবহার করে যাচাই করতে যে একটা নির্দিষ্ট user_id সত্যিই অস্তিত্বশীল কিনা

---

## ৪. Product Service — বিস্তারিত বিশ্লেষণ

**Product Service** প্রোডাক্টের তথ্য ব্যবস্থাপনার দায়িত্বে থাকে, এবং **MongoDB** ব্যবহার করে। এই সার্ভিসের একটা বিশেষত্ব হলো — একটা নতুন প্রোডাক্ট তৈরি করলেই এটা **স্বয়ংক্রিয়ভাবে** Inventory Service-এ একটা inventory রেকর্ড তৈরি করিয়ে দেয় (এটাই "product-inventory integration")।

**চিত্র ৩**-এ এই সার্ভিসের গঠন দেখা যায় (Port 8000-এ চলে):
- **API Endpoints** — প্রোডাক্ট তৈরি, তালিকা দেখা, নির্দিষ্ট বিবরণ, আপডেট, category তালিকা
- **Product Logic** — CRUD অপারেশন, ফিল্টারিং/সার্চ, category ব্যবস্থাপনা
- **Service Integration** — Inventory তৈরি করা, রিট্রাই লজিক, এরর হ্যান্ডলিং

### MongoDB Document গঠন

```json
{
    "_id": ObjectId,
    "name": str,
    "description": str,
    "category": str,
    "price": float,
    "quantity": int
}
```

**MongoDB-তে `_id`** স্বয়ংক্রিয়ভাবে তৈরি হয়, যেটাকে বলে `ObjectId` — এটা প্রতিটা document-এর জন্য একটা ইউনিক পরিচয়পত্র (identifier)।

### API Routes

- `POST /api/v1/products/` → নতুন প্রোডাক্ট তৈরি করা (সাথে সাথে ইনভেন্টরি অটো-তৈরি হয়)
- `GET /api/v1/products/` → ফিল্টার এবং পেজিনেশন সহ প্রোডাক্টের তালিকা দেখা
- `GET /api/v1/products/{product_id}` → নির্দিষ্ট একটা প্রোডাক্টের বিস্তারিত দেখা
- `PUT /api/v1/products/{product_id}` → প্রোডাক্টের তথ্য আংশিক আপডেট করা
- `DELETE /api/v1/products/{product_id}` → প্রোডাক্ট মুছে ফেলা
- `GET /api/v1/products/category/list` → সব ইউনিক category-র তালিকা দেখা

**পেজিনেশন (Pagination) কী?** যখন অনেক বেশি ডেটা থাকে (যেমন হাজারো প্রোডাক্ট), তখন সবগুলো একসাথে না পাঠিয়ে ছোট ছোট "পাতায়" (page) ভাগ করে পাঠানো হয় — যেমন প্রতিবার শুধু ১০টা প্রোডাক্ট দেখানো। এতে সার্ভার এবং নেটওয়ার্কের উপর চাপ কম পড়ে।

---

## ৫. Inventory Service — বিস্তারিত বিশ্লেষণ

**Inventory Service** স্টক (মজুদ) পরিমাণ, রিজার্ভেশন, এবং স্টকের ইতিহাস ট্র্যাক করার দায়িত্বে থাকে। এটা **PostgreSQL** ব্যবহার করে, সাথে বিস্তারিত ইতিহাস ট্র্যাকিং সিস্টেম আছে।

**চিত্র ৪**-এ এই সার্ভিসের গঠন দেখা যায় (Port 8002-এ চলে):
- **API Endpoints** — ইনভেন্টরি তৈরি, চেক করা, রিজার্ভ করা, রিলিজ করা
- **Inventory Logic** — স্টক ব্যবস্থাপনা, রিজার্ভেশন সিস্টেম, কম স্টক শনাক্তকরণ, ইতিহাস ট্র্যাকিং
- **Service Integration** — প্রোডাক্ট যাচাই, নোটিফিকেশন এলার্ট, রিট্রাই লজিক

### ডেটাবেজ মডেল

**Inventory Items টেবিল:**

```python
class InventoryItem(Base):
    __tablename__ = "inventory_items"
    
    id = Column(Integer, primary_key=True, index=True)
    product_id = Column(String, unique=True, index=True, nullable=False)
    available_quantity = Column(Integer, nullable=False, default=0)
    reserved_quantity = Column(Integer, nullable=False, default=0)
    reorder_threshold = Column(Integer, nullable=False, default=5)
    
    __table_args__ = (
        CheckConstraint('available_quantity >= 0'),
        CheckConstraint('reserved_quantity >= 0'),
    )
```

এখানে দুটো গুরুত্বপূর্ণ ধারণা:
- **`available_quantity`** বনাম **`reserved_quantity`** — এটা একটা চমৎকার ডিজাইন প্যাটার্ন। যখন কেউ অর্ডার দেয়, তখন সাথে সাথে স্টক থেকে বাদ না দিয়ে, বরং সেই পরিমাণ **"reserved" (সংরক্ষিত)** হিসেবে চিহ্নিত করা হয়। এভাবে available quantity আর reserved quantity আলাদা রাখলে বোঝা যায় ঠিক কতটা সত্যিকারভাবে বিক্রয়যোগ্য আছে, আর কতটা ইতিমধ্যে কারো অর্ডারে "বুকড" হয়ে আছে।
- **`CheckConstraint('available_quantity >= 0')`** → এটা ডেটাবেজ-লেভেলে একটা নিয়ম বেঁধে দেয় যে quantity কখনো **নেগেটিভ (ঋণাত্মক)** হতে পারবে না — এটা ডেটা সঠিক রাখার একটা নিরাপত্তা ব্যবস্থা, ভুল কোডের কারণেও যাতে ডেটাবেজে অসংগতি তৈরি না হয়।
- **`reorder_threshold`** → এটা একটা সীমা, যার নিচে স্টক নেমে গেলে বুঝতে হবে আবার নতুন করে স্টক অর্ডার (reorder) করার সময় হয়েছে

**Inventory History টেবিল:**

```python
class InventoryHistory(Base):
    __tablename__ = "inventory_history"
    
    id = Column(Integer, primary_key=True, index=True)
    product_id = Column(String, index=True, nullable=False)
    quantity_change = Column(Integer, nullable=False)
    previous_quantity = Column(Integer, nullable=False)
    new_quantity = Column(Integer, nullable=False)
    change_type = Column(String, nullable=False)  # "add", "remove", "reserve", "release"
    reference_id = Column(String, nullable=True)
    timestamp = Column(DateTime(timezone=True), server_default=func.now())
```

এই টেবিলটা প্রতিটা ইনভেন্টরি পরিবর্তনের একটা সম্পূর্ণ **অডিট ট্রেইল (audit trail)** রাখে — কবে, কী কারণে (add/remove/reserve/release), কত পরিমাণ পরিবর্তন হয়েছে, তার সবকিছুর রেকর্ড। এটা ব্যবসায়িকভাবে খুবই গুরুত্বপূর্ণ, কারণ ভবিষ্যতে কোনো সমস্যা হলে (যেমন স্টকের হিসাব না মিললে) এই history থেকে ঠিক কী কী পরিবর্তন হয়েছিল তা ট্র্যাক করা যায়।

### API Routes

- `POST /api/v1/inventory/` → নতুন ইনভেন্টরি রেকর্ড তৈরি (validation এবং history-সহ)
- `GET /api/v1/inventory/` → ফিল্টার/পেজিনেশনসহ ইনভেন্টরি তালিকা
- `GET /api/v1/inventory/check` → নির্দিষ্ট পরিমাণের জন্য পর্যাপ্ত স্টক আছে কিনা চেক করা
- `GET /api/v1/inventory/{product_id}` → নির্দিষ্ট প্রোডাক্টের সম্পূর্ণ ইনভেন্টরি বিবরণ
- `PUT /api/v1/inventory/{product_id}` → ইনভেন্টরি পরিমাণ/সেটিংস আপডেট (স্বয়ংক্রিয় লগিং সহ)
- `POST /api/v1/inventory/reserve` → অর্ডার প্রসেসিং-এর জন্য ইনভেন্টরি রিজার্ভ করা
- `POST /api/v1/inventory/release` → আগে রিজার্ভ করা ইনভেন্টরি আবার available স্টকে ফিরিয়ে আনা
- `POST /api/v1/inventory/adjust` → কারণ-সহ ম্যানুয়াল ইনভেন্টরি সমন্বয়
- `GET /api/v1/inventory/low-stock` → reorder threshold-এর নিচে থাকা প্রোডাক্টের তালিকা
- `GET /api/v1/inventory/history/{product_id}` → নির্দিষ্ট প্রোডাক্টের সম্পূর্ণ পরিবর্তনের ইতিহাস

---

## ৬. Order Service — বিস্তারিত বিশ্লেষণ

**Order Service** সম্পূর্ণ অর্ডার জীবনচক্র (lifecycle) ব্যবস্থাপনার দায়িত্বে থাকে, এবং **MongoDB** ব্যবহার করে। এই সার্ভিসটা বিশেষভাবে গুরুত্বপূর্ণ, কারণ এটা **User, Product, এবং Inventory** — তিনটা সার্ভিসের সাথেই সরাসরি যোগাযোগ করে অর্ডার প্রসেস করে।

**চিত্র ৫**-এ এই সার্ভিসের গঠন দেখা যায় (Port 8001-এ চলে):
- **API Endpoints** — অর্ডার তৈরি, তালিকা, status আপডেট, ডিলিট
- **Order Logic** — অর্ডার তৈরি, status ব্যবস্থাপনা, দাম হিসাব, business rule
- **Service Orchestration** — মাল্টি-সার্ভিস যাচাই, ইনভেন্টরি ব্যবস্থাপনা, transaction logic

**Orchestration (অর্কেস্ট্রেশন) কী?** এটা এমন একটা পদ্ধতি, যেখানে একটা সার্ভিস (এখানে Order Service) একাধিক অন্য সার্ভিসের সাথে সমন্বয় করে একটা জটিল কাজ সম্পন্ন করে। যেমন অর্ডার তৈরি করতে হলে Order Service-কে প্রথমে User Service দিয়ে ইউজার যাচাই করতে হয়, Product Service দিয়ে প্রোডাক্ট তথ্য নিতে হয়, আর Inventory Service দিয়ে স্টক রিজার্ভ করতে হয় — এই পুরো প্রক্রিয়াটা "অর্কেস্ট্রেট" করে Order Service।

### MongoDB Document গঠন (Order)

```json
{
    "_id": ObjectId,
    "user_id": string,
    "items": [
        {
            "product_id": string,
            "quantity": integer,
            "price": decimal
        }
    ],
    "total_price": decimal,
    "status": string,
    "shipping_address": { ... },
    "created_at": datetime,
    "updated_at": datetime
}
```

লক্ষণীয়:
- `"items"` একটা **array (তালিকা)**, কারণ একটা অর্ডারে একাধিক প্রোডাক্ট থাকতে পারে
- `"price"` প্রতিটা item-এর সাথে আলাদা করে সেভ করা হয় — এটা গুরুত্বপূর্ণ কারণ **অর্ডারের সময়কার দাম** সংরক্ষণ করা দরকার, ভবিষ্যতে প্রোডাক্টের দাম বদলে গেলেও পুরনো অর্ডারের হিসাব যেন ঠিক থাকে
- `"status"` — অর্ডারের বর্তমান অবস্থা (pending, paid, cancelled ইত্যাদি)

### API Routes

- `POST /api/v1/orders/` → সম্পূর্ণ validation, inventory reservation, এবং ব্যর্থ হলে rollback সহ নতুন অর্ডার তৈরি
- `GET /api/v1/orders/` → status, user, তারিখের রেঞ্জ দিয়ে ফিল্টার করে পেজিনেটেড অর্ডার তালিকা
- `GET /api/v1/orders/{order_id}` → নির্দিষ্ট একটা অর্ডারের বিস্তারিত তথ্য
- `GET /api/v1/orders/user/{user_id}` → নির্দিষ্ট ইউজারের সব অর্ডার (status ফিল্টার এবং পেজিনেশনসহ)
- `PUT /api/v1/orders/{order_id}/status` → validation এবং ইনভেন্টরির উপর প্রভাব হ্যান্ডলিং সহ অর্ডারের status আপডেট
- `DELETE /api/v1/orders/{order_id}` → যোগ্য হলে অর্ডার বাতিল করা এবং স্বয়ংক্রিয়ভাবে রিজার্ভ করা ইনভেন্টরি ছেড়ে দেওয়া

**Rollback কী?** যদি অর্ডার তৈরি করার প্রক্রিয়ার মাঝপথে কোনো সমস্যা হয় (যেমন ইনভেন্টরি রিজার্ভ করা যায়নি), তাহলে ইতিমধ্যে করা সব পরিবর্তন **বাতিল (undo)** করে দেওয়া হয়, যাতে সিস্টেম একটা অসম্পূর্ণ বা ভুল অবস্থায় আটকে না থাকে। এটা ডেটা সঠিক (consistent) রাখার জন্য খুবই গুরুত্বপূর্ণ।

---

## ৭. অ্যাপ্লিকেশন চালু করা

পুরো সিস্টেম একসাথে চালু করতে `docker-compose` ব্যবহার করা হয়:

```bash
docker-compose up --build -d
```

- `--build` → চালু করার আগে সব image নতুন করে বিল্ড করবে (যদি কোনো কোড পরিবর্তন হয়ে থাকে)
- `-d` → detached mode, মানে সব সার্ভিস ব্যাকগ্রাউন্ডে চলবে

এবার JSON আউটপুট সুন্দরভাবে দেখার জন্য `jq` টুল ইনস্টল করি:

```bash
apt-get update && apt-get install -y jq
```

**`jq` কী?** এটা একটা কমান্ড-লাইন টুল, যেটা দিয়ে JSON ডেটা সুন্দরভাবে, সাজানো (formatted) আকারে দেখা যায় — সাধারণ JSON টেক্সট একলাইনে জট পাকিয়ে থাকলে পড়া কঠিন হয়, `jq` সেটাকে ইন্ডেন্টেশনসহ সুন্দর করে দেখায়।

---

## ৮. টেস্টিং ফ্লো-এর সংক্ষিপ্ত চিত্র

আমরা নিচের ধাপগুলো অনুসরণ করে পুরো সিস্টেম টেস্ট করব:

1. ইউজার রেজিস্ট্রেশন এবং authentication
2. ইউজারের ঠিকানা যোগ করা
3. প্রোডাক্ট তৈরি করা (ইনভেন্টরি স্বয়ংক্রিয়ভাবে তৈরি হবে)
4. প্রোডাক্ট ব্রাউজ করা এবং ইনভেন্টরি চেক করা
5. অর্ডার দেওয়া
6. অর্ডারের বিবরণ দেখা এবং ব্যবস্থাপনা করা

---

## ৯. ধাপ ১: ইউজার রেজিস্ট্রেশন এবং Authentication

### ১.১ নতুন ইউজার রেজিস্টার করা

```bash
curl -X POST "http://localhost/api/v1/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "customer@example.com",
    "password": "Password123",
    "first_name": "Example",
    "last_name": "Customer",
    "phone": "555-123-4567"
  }' | jq .
```

- `curl -X POST` → একটা HTTP `POST` রিকোয়েস্ট পাঠাচ্ছি, যেটা নতুন কিছু তৈরি করার জন্য ব্যবহার হয়
- `-H "Content-Type: application/json"` → বলছি, আমরা যে ডেটা পাঠাচ্ছি সেটা JSON ফরম্যাটে
- `-d '{...}'` → এটা হলো আসল ডেটা (data/body), যেটা সার্ভারে পাঠানো হচ্ছে
- `| jq .` → রেসপন্সটা `jq`-এর মধ্য দিয়ে পাইপ করে সুন্দরভাবে ফরম্যাট করে দেখানো হচ্ছে

**গুরুত্বপূর্ণ পয়েন্ট:** রেসপন্সে `password` ফিল্ডটা দেখানো হয় না (নিরাপত্তার কারণে), এবং `addresses: []` দেখাচ্ছে কারণ এখনো কোনো ঠিকানা যোগ করা হয়নি।

### ১.২ Login করে Authentication Token নেওয়া

```bash
curl -X POST "http://localhost/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=customer@example.com&password=Password123" | jq .
```

লক্ষ্য করো এখানে Content-Type ভিন্ন — `application/x-www-form-urlencoded`, যেটা সাধারণত লগইন ফর্মের জন্য ব্যবহার হয় (JSON না)।

রেসপন্সে দুটো টোকেন পাওয়া যাবে:
- `access_token` → এটা প্রতিটা পরবর্তী রিকোয়েস্টে পাঠাতে হবে, যাতে সার্ভার জানে আমরা কে
- `refresh_token` → `access_token`-এর মেয়াদ শেষ হয়ে গেলে, এই টোকেন দিয়ে নতুন `access_token` নেওয়া যায়, আবার লগইন করতে হয় না

এই টোকেনটা একটা ভেরিয়েবলে সেভ করে রাখি, যাতে পরবর্তী কমান্ডগুলোতে সহজে ব্যবহার করা যায়:

```bash
TOKEN="eyJhbGciOiJS..."
```

### ১.৩ Authentication যাচাই করা

```bash
curl -X GET "http://localhost/api/v1/users/me" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

- `-H "Authorization: Bearer $TOKEN"` → এটাই হলো **JWT authentication**-এর মূল পদ্ধতি। `Bearer` শব্দটা একটা প্রমিত (standard) শব্দ, যার মানে "এই টোকেন বহনকারী (bearer)-কেই বিশ্বাস করো"। সার্ভার এই টোকেনটা যাচাই করে বুঝে নেয় রিকোয়েস্টটা কার পক্ষ থেকে আসছে।

---

## ১০. ধাপ ২: ইউজারের ঠিকানা যোগ করা

```bash
curl -X POST "http://localhost/api/v1/users/me/addresses" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country",
    "is_default": true
  }' | jq .
```

লক্ষ্য করো, এবার আমরা `Authorization` হেডারও পাঠাচ্ছি, কারণ এই endpoint-টা লগইন করা ইউজারের জন্য — সার্ভারকে জানতে হবে ঠিক কোন ইউজারের ঠিকানা যোগ করা হচ্ছে।

---

## ১১. ধাপ ৩: স্বয়ংক্রিয় ইনভেন্টরিসহ প্রোডাক্ট তৈরি করা

এবার আমরা তিনটা ভিন্ন প্রোডাক্ট তৈরি করব। আমাদের ইন্টিগ্রেশনের কারণে, প্রতিটা প্রোডাক্টের জন্য **স্বয়ংক্রিয়ভাবে** ইনভেন্টরি রেকর্ড তৈরি হবে।

### ৩.১ প্রোডাক্ট ১ — স্মার্টফোন

```bash
curl -X POST "http://localhost/api/v1/products/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Premium Smartphone",
    "description": "Latest model with high-end camera and long battery life",
    "category": "Electronics",
    "price": 899.99,
    "quantity": 50
  }' | jq .
```

রেসপন্স থেকে `_id`-টা সেভ করে রাখি (এটা এই প্রোডাক্টকে পরবর্তীতে চিহ্নিত করতে দরকার হবে):

```bash
PRODUCT_ID_1="product_id_1"
```

### ৩.২ ও ৩.৩ — আরও দুটো প্রোডাক্ট

একই পদ্ধতিতে **Wireless Headphones** আর **Smart Fitness Watch** তৈরি করা হয়, শুধু ডেটা ভিন্ন। প্রতিটার `_id` আলাদা ভেরিয়েবলে সেভ করা হয় (`PRODUCT_ID_2`, `PRODUCT_ID_3`)।

---

## ১২. ধাপ ৪: প্রোডাক্ট ব্রাউজ এবং ইনভেন্টরি চেক করা

### ৪.১ সব প্রোডাক্ট দেখা

```bash
curl -X GET "http://localhost/api/v1/products/" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### ৪.২ ইনভেন্টরি ঠিকমতো তৈরি হয়েছে কিনা যাচাই করা

```bash
curl -X GET "http://localhost/api/v1/inventory/$PRODUCT_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

রেসপন্সে দেখা যাবে `available_quantity: 50` (Product তৈরির সময় দেওয়া `quantity`-র সমান) এবং `reserved_quantity: 0` (এখনো কোনো অর্ডার হয়নি)। এটাই নিশ্চিত করে যে **product-inventory integration** ঠিকমতো কাজ করছে — প্রোডাক্ট তৈরি করলেই আলাদা করে ইনভেন্টরি তৈরি করতে হয়নি, এটা নিজে থেকেই হয়ে গেছে।

### ৪.৩ Category দিয়ে ফিল্টার করা

```bash
curl -X GET "http://localhost/api/v1/products/?category=Electronics" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

`?category=Electronics` হলো একটা **query parameter**, যেটা URL-এর সাথে যুক্ত করে বলা হয় শুধু নির্দিষ্ট category-র প্রোডাক্ট দেখাতে।

### ৪.৪ দামের রেঞ্জ দিয়ে ফিল্টার করা

```bash
curl -X GET "http://localhost/api/v1/products/?min_price=100&max_price=300" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

এখানে দুটো query parameter একসাথে (`min_price` এবং `max_price`) ব্যবহার করা হয়েছে, `&` চিহ্ন দিয়ে যুক্ত করে।

---

## ১৩. ধাপ ৫: অর্ডার দেওয়া

### ৫.১ ইউজার ID বের করা

```bash
curl -X GET "http://localhost/api/v1/users/me" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

```bash
USER_ID="1"
```

### ৫.২ একটা প্রোডাক্টের জন্য অর্ডার দেওয়া

```bash
curl -X POST "http://localhost/api/v1/orders/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "'$USER_ID'",
    "items": [
      {
        "product_id": "'$PRODUCT_ID_1'",
        "quantity": 1,
        "price": 899.99
      }
    ],
    "shipping_address": { ... }
  }' | jq .
```

লক্ষ্য করো, এখানে `"'$USER_ID'"` এর মতো লেখা হয়েছে — এটা একটা bash কৌশল, যেখানে single quote-এর মধ্যে দিয়ে ভেরিয়েবলকে সরাসরি JSON স্ট্রিং-এর ভেতরে বসানো হচ্ছে। এভাবে আগে সেভ করা `USER_ID` আর `PRODUCT_ID_1` ভেরিয়েবলের আসল মান JSON-এ বসে যায়।

Order তৈরি হওয়ার পর response-এ `status: "pending"` দেখাবে (নতুন অর্ডার সবসময় "pending" স্ট্যাটাসে শুরু হয়) এবং `total_price` স্বয়ংক্রিয়ভাবে হিসাব করে দেখাবে।

```bash
ORDER_ID_1="order_id_1"
```

### ৫.৩ একাধিক প্রোডাক্টের জন্য অর্ডার দেওয়া

একইভাবে, কিন্তু এবার `items` array-তে দুটো ভিন্ন প্রোডাক্ট থাকবে। লক্ষ্য করো, `total_price` স্বয়ংক্রিয়ভাবে দুটো item-এর দামের যোগফল (249.99 + 179.99×2 = 609.97) হিসাব করে দেখাচ্ছে।

---

## ১৪. ধাপ ৬: অর্ডার দেখা এবং ব্যবস্থাপনা করা

### ৬.১ নির্দিষ্ট অর্ডারের বিবরণ দেখা

```bash
curl -X GET "http://localhost/api/v1/orders/$ORDER_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### ৬.২ সব অর্ডারের তালিকা দেখা

```bash
curl -X GET "http://localhost/api/v1/orders/" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### ৬.৩ অর্ডারের Status আপডেট করা

```bash
curl -X PUT "http://localhost/api/v1/orders/$ORDER_ID_1/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "paid"
  }' | jq .
```

`curl -X PUT` → HTTP `PUT` মেথড ব্যবহার হয় যখন কোনো **বিদ্যমান রিসোর্স আপডেট** করতে হয় (POST নতুন কিছু তৈরির জন্য, PUT পুরনো কিছু পরিবর্তনের জন্য)।

### ৬.৪ অর্ডার বাতিল (Cancel) করা

```bash
curl -X DELETE "http://localhost/api/v1/orders/$ORDER_ID_2" \
  -H "Authorization: Bearer $TOKEN"
```

এই কমান্ডটা **`204 No Content`** স্ট্যাটাস রিটার্ন করে, কোনো response body ছাড়াই। **`204` মানে কী?** এটা একটা HTTP স্ট্যাটাস কোড, যার মানে হলো "রিকোয়েস্টটা সফল হয়েছে, কিন্তু তোমাকে ফেরত দেওয়ার মতো কোনো কনটেন্ট নেই" — DELETE অপারেশনের জন্য এটা খুবই সাধারণ, কারণ জিনিসটাই তো মুছে গেছে, তার সম্পর্কে আর কিছু দেখানোর দরকার নেই।

আবার চেক করে দেখি অর্ডার সত্যিই বাতিল হয়েছে কিনা — `status: "cancelled"` দেখা যাবে।

### ৬.৫ অর্ডার-পরবর্তী ইনভেন্টরি পরিবর্তন যাচাই করা

Order দেওয়ার পর Product 1-এর ইনভেন্টরি চেক করলে দেখা যাবে:

```json
{
  "available_quantity": 49,
  "reserved_quantity": 1
}
```

লক্ষ্য করো — `available_quantity` কমে ৪৯ হয়েছে (৫০ থেকে ১ কম), আর `reserved_quantity` বেড়ে ১ হয়েছে। এটাই দেখায় Order Service ঠিকমতো Inventory Service-এর সাথে যোগাযোগ করে স্টক **reserve** করে ফেলেছে।

আবার, Order বাতিল করার পর Product 2-এর ইনভেন্টরি চেক করলে দেখা যাবে `available_quantity` আগের মতোই ফিরে এসেছে (`100`) এবং `reserved_quantity` আবার `0` — এটা প্রমাণ করে অর্ডার বাতিল হলে reserved স্টক স্বয়ংক্রিয়ভাবে available-এ ফিরে আসে।

---

## ১৫. সম্পূর্ণ Workflow যাচাই

পুরো টেস্টিং প্রক্রিয়া শেষে আমরা নিশ্চিত হতে পারি যে নিচের সবগুলো ধাপ সফলভাবে কাজ করেছে:

- ✅ ইউজার একাউন্ট রেজিস্টার এবং authenticate করা হয়েছে
- ✅ শিপিং ঠিকানা যোগ করা হয়েছে
- ✅ তিনটা ভিন্ন প্রোডাক্ট তৈরি করা হয়েছে, স্বয়ংক্রিয় ইনভেন্টরি তৈরিসহ
- ✅ প্রতিটা প্রোডাক্টের জন্য ইনভেন্টরি তৈরি হয়েছে তা যাচাই করা হয়েছে
- ✅ প্রোডাক্ট ব্রাউজ এবং ফিল্টার করা হয়েছে
- ✅ প্রোডাক্টের জন্য অর্ডার দেওয়া হয়েছে
- ✅ অর্ডারের বিবরণ দেখা হয়েছে
- ✅ অর্ডারের status আপডেট করা হয়েছে
- ✅ একটা অর্ডার বাতিল করা হয়েছে
- ✅ অর্ডার-অপারেশনের পর ইনভেন্টরির পরিবর্তন যাচাই করা হয়েছে

এটা নিশ্চিত করে যে সব microservice একসাথে সঠিকভাবে কাজ করছে, এবং সিস্টেমের মধ্য দিয়ে ডেটা প্রত্যাশিতভাবে প্রবাহিত (flow) হচ্ছে।

---

## ১৬. সংক্ষেপে — মূল ধারণাগুলো (Summary)

| ধারণা | ব্যাখ্যা |
|---|---|
| **Microservices** | বড় অ্যাপ্লিকেশনকে ছোট ছোট স্বাধীন সার্ভিসে ভাগ করে বানানোর পদ্ধতি |
| **API Gateway (Nginx)** | Client-এর রিকোয়েস্ট সঠিক সার্ভিসে রুট করে দেয় |
| **FastAPI** | Python-এর দ্রুতগতির ওয়েব ফ্রেমওয়ার্ক, API বানানোর জন্য ব্যবহৃত |
| **JWT Token** | লগইন-পরবর্তী পরিচয় যাচাইয়ের জন্য ব্যবহৃত সুরক্ষিত টোকেন |
| **PostgreSQL vs MongoDB** | Relational (SQL) বনাম Document-based (NoSQL) ডেটাবেজ |
| **Available vs Reserved Quantity** | স্টকের কতটা সত্যিকার বিক্রয়যোগ্য, আর কতটা অর্ডারে বুকড আছে তার আলাদা হিসাব |
| **Service Orchestration** | একটা সার্ভিস একাধিক অন্য সার্ভিসের সাথে সমন্বয় করে জটিল কাজ সম্পন্ন করা |
| **Rollback** | মাঝপথে সমস্যা হলে আগের পরিবর্তন বাতিল করে সিস্টেমকে সঠিক অবস্থায় ফিরিয়ে আনা |
| `curl -X POST/GET/PUT/DELETE` | যথাক্রমে তৈরি, পড়া, আপডেট, এবং মুছে ফেলার জন্য HTTP মেথড |
| **Bearer Token (Authorization header)** | রিকোয়েস্টের সাথে identity প্রমাণের জন্য টোকেন পাঠানোর প্রমিত পদ্ধতি |
| `jq` | JSON আউটপুট সুন্দরভাবে ফরম্যাট করে দেখানোর কমান্ড-লাইন টুল |
| **204 No Content** | সফল অপারেশন, কিন্তু ফেরত দেওয়ার মতো কোনো ডেটা নেই (যেমন ডিলিটের পর) |

**পুরো সিস্টেমটার সারমর্ম:**
একজন ইউজার রেজিস্টার করে, লগইন করে টোকেন পায় → ঠিকানা যোগ করে → প্রোডাক্ট তৈরি করলে অটোমেটিক ইনভেন্টরি তৈরি হয় → ইউজার প্রোডাক্ট ব্রাউজ করে অর্ডার দেয় → Order Service, Inventory Service-কে বলে স্টক রিজার্ভ করতে → অর্ডার status পরিবর্তন করা যায় (paid), বা বাতিল করলে রিজার্ভড স্টক আবার available-এ ফিরে আসে। পুরো প্রক্রিয়াটাই চারটা স্বাধীন সার্ভিস একে অপরের সাথে সমন্বয় করে সম্পন্ন করে — এটাই আধুনিক microservices architecture-এর একটা বাস্তব, সম্পূর্ণ উদাহরণ।

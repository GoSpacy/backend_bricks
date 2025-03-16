## Foundational Backend Plan

### 1. **Directory Structure**

```
ad_space_app/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app entry point
│   ├── models/                  # Data models (ORM models and Pydantic models)
│   │   ├── __init__.py
│   │   ├── ad_space.py          # AdSpace ORM model
│   │   ├── media_type.py        # MediaType ORM model
│   │   ├── ad_metric.py         # AdMetric ORM model
│   │   ├── pydantic_models.py   # Pydantic models for input/output validation
│   ├── services/                # Business logic layer
│   │   ├── __init__.py
│   │   ├── ad_space_service.py  # Handles ad space-related operations
│   │   ├── media_type_service.py# Handles media type-related operations
│   │   ├── ad_metric_service.py # Handles metrics-related operations
│   ├── repositories/            # Data access layer (Repository pattern)
│   │   ├── __init__.py
│   │   ├── ad_space_repository.py
│   │   ├── media_type_repository.py
│   │   ├── ad_metric_repository.py
│   ├── validators/              # Validation layer
│   │   ├── __init__.py
│   │   ├── ad_space_validator.py
│   ├── api/                     # API routes (controllers)
│   │   ├── __init__.py
│   │   ├── ad_space_routes.py   # AdSpace related API endpoints
│   │   ├── media_type_routes.py # MediaType related API endpoints
│   │   ├── ad_metric_routes.py  # AdMetric related API endpoints
│   ├── utils/                   # Utility functions and helpers
│   │   ├── __init__.py
│   │   ├── logger.py            # Logger utility
│   │   ├── config.py            # Configuration settings
├── requirements.txt             # List of project dependencies
├── .env                         # Environment variables
├── alembic.ini                  # Alembic configuration (for database migrations)
└── README.md                    # Project description
```

---

### 2. **Directory Breakdown**

#### `app/main.py` (Entry Point)
- This file contains the **FastAPI app** initialization.
- Here, you will include the application setup, such as including the routers, middleware, and dependencies.

```python
from fastapi import FastAPI
from .api import ad_space_routes, media_type_routes, ad_metric_routes

app = FastAPI()

# Include routers
app.include_router(ad_space_routes.router)
app.include_router(media_type_routes.router)
app.include_router(ad_metric_routes.router)
```

#### `app/models/` (Data Models and Pydantic Models)
- **ORM Models**: These are SQLAlchemy or Tortoise ORM models that represent the database schema.
- **Pydantic Models**: These models are used for request/response validation in FastAPI.

For example, `ad_space.py` (ORM model) and `pydantic_models.py` (Pydantic models) will look like this:

**`models/ad_space.py`**:
```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from .database import Base

class AdSpace(Base):
    __tablename__ = 'ad_spaces'
    id = Column(Integer, primary_key=True, index=True)
    campaign_name = Column(String)
    duration = Column(String)
    impressions = Column(Integer)
    clicks = Column(Integer)
    media_type_id = Column(Integer, ForeignKey('media_types.id'))

    media_type = relationship("MediaType", back_populates="ad_spaces")
```

**`models/pydantic_models.py`**:
```python
from pydantic import BaseModel

class AdSpaceBase(BaseModel):
    campaign_name: str
    duration: str
    impressions: int
    clicks: int

class AdSpaceCreate(AdSpaceBase):
    pass

class AdSpaceResponse(AdSpaceBase):
    id: int

    class Config:
        orm_mode = True
```

#### `app/services/` (Business Logic Layer)
- The **service layer** handles the core business logic, where most of the functionality resides. This is where you implement the SOLID principles (i.e., each class should have a single responsibility).
  
For example, `ad_space_service.py` will look like this:

```python
from typing import List
from app.models.ad_space import AdSpace
from app.repositories.ad_space_repository import AdSpaceRepository
from app.validators.ad_space_validator import AdSpaceValidator

class AdSpaceService:
    def __init__(self, repository: AdSpaceRepository, validator: AdSpaceValidator):
        self.repository = repository
        self.validator = validator

    def create_ad_space(self, ad_space_data) -> AdSpace:
        if not self.validator.is_valid(ad_space_data):
            raise ValueError("Invalid Ad Space Data")
        return self.repository.create(ad_space_data)

    def get_all_ad_spaces(self) -> List[AdSpace]:
        return self.repository.get_all()
```

#### `app/repositories/` (Data Access Layer)
- The **repository pattern** helps separate data access logic from business logic. Each repository is responsible for interacting with the database and providing data to the service layer.

For example, `ad_space_repository.py` will look like this:

```python
from app.models.ad_space import AdSpace
from sqlalchemy.orm import Session

class AdSpaceRepository:
    def __init__(self, db: Session):
        self.db = db

    def create(self, ad_space_data):
        db_ad_space = AdSpace(**ad_space_data)
        self.db.add(db_ad_space)
        self.db.commit()
        self.db.refresh(db_ad_space)
        return db_ad_space

    def get_all(self):
        return self.db.query(AdSpace).all()
```

#### `app/validators/` (Validation Layer)
- This layer is responsible for validating incoming data before it reaches the business logic.

For example, `ad_space_validator.py` will look like this:

```python
class AdSpaceValidator:
    def is_valid(self, ad_space_data):
        # Perform validation logic here
        if ad_space_data['impressions'] < 0:
            return False
        return True
```

#### `app/api/` (API Routes)
- The **API layer** handles incoming HTTP requests and maps them to appropriate service methods. This is where FastAPI's routing and dependency injection come in.
  
For example, `ad_space_routes.py`:

```python
from fastapi import APIRouter, Depends
from app.services.ad_space_service import AdSpaceService
from app.repositories.ad_space_repository import AdSpaceRepository
from app.validators.ad_space_validator import AdSpaceValidator
from app.models.pydantic_models import AdSpaceCreate, AdSpaceResponse
from app.database import get_db

router = APIRouter()

@router.post("/adspaces", response_model=AdSpaceResponse)
def create_ad_space(ad_space: AdSpaceCreate, db: Session = Depends(get_db)):
    repository = AdSpaceRepository(db)
    validator = AdSpaceValidator()
    service = AdSpaceService(repository, validator)
    return service.create_ad_space(ad_space.dict())
```

#### `app/utils/` (Utilities)
- This folder contains utility functions, helpers, and configuration settings, such as logging, database connections, and configuration management.

For example, `logger.py`:

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Create a file handler
handler = logging.FileHandler("app.log")
handler.setLevel(logging.INFO)

formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formatter)

logger.addHandler(handler)
```

---

### 3. **Putting It All Together**

- Each module adheres to **SOLID** principles:
  - **Single Responsibility Principle (SRP)**: Each module has a single responsibility, such as handling database interactions or validating data.
  - **Open/Closed Principle (OCP)**: You can easily extend functionality without modifying existing code (e.g., adding new ad types without changing the core service logic).
  - **Liskov Substitution Principle (LSP)**: You can substitute different implementations of the repositories or services without breaking the application.
  - **Interface Segregation Principle (ISP)**: Different classes (e.g., `AdSpaceRepository`, `AdSpaceValidator`) expose only the necessary methods.
  - **Dependency Inversion Principle (DIP)**: Dependencies like `AdSpaceRepository` and `AdSpaceService` are injected into other classes, making them easy to mock or swap.

---

### 4. **Advantages of This Structure**
- **Modularity**: The app is divided into well-defined layers and responsibilities.
- **Scalability**: You can scale each component independently, such as adding more services or repositories.
- **Maintainability**: The code is easier to maintain, test, and extend with new features.

---

Let me know if you need further clarification or assistance with implementing the solution!

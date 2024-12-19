# Реалізація інформаційного та програмного забезпечення

В рамках проекту розробляється:

- SQL-скрипт для створення та початкового наповнення бази даних
- RESTful сервіс для управління даними

## SQL

```sql
-- Створення бази даних
CREATE DATABASE IF NOT EXISTS `mydb` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE `mydb`;

-- Таблиця Category
CREATE TABLE `Category` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `description` VARCHAR(1000) NULL
) ENGINE=InnoDB;

-- Таблиця Data
CREATE TABLE `Data` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL UNIQUE,
    `content` VARCHAR(2000) NOT NULL,
    `upload_date` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `last_edit_date` DATETIME NULL,
    `category_id` INT NOT NULL,
    FOREIGN KEY (`category_id`) REFERENCES `Category` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB;


INSERT INTO Data (id, name, content, upload_date, category_id) VALUES (1, 'Test Data', 'Test Content', NOW(), 1);
INSERT INTO Category (id, name, description) VALUES (1, 'Sample Category', 'Test description');




Python

Файл main.py


from fastapi import FastAPI
from database import engine, Base
from routes import router


Base.metadata.create_all(bind=engine)
app = FastAPI()
app.include_router(router)


Файл database.py


from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from config import mypassword

MY_DATABASE_URL = f"mysql+pymysql://root:{mypassword}@127.0.0.1:3306/mydb"

engine = create_engine(MY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


Файл models.py

from sqlalchemy import Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from database import Base

class Category(Base):
    __tablename__ = 'categories'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(255), nullable=False)
    description = Column(String(1000), nullable=True)

    data_items = relationship("Data", back_populates="category")

class Data(Base):
    __tablename__ = 'data'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(255), nullable=False, unique=True)
    content = Column(String(2000), nullable=False)
    upload_date = Column(DateTime, nullable=False)
    last_edit_date = Column(DateTime, nullable=True)
    category_id = Column(Integer, ForeignKey('categories.id'), nullable=False)

    category = relationship("Category", back_populates="data_items")

Файл routes.py

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from models import Data, Category
from schemas import DataCreate, CategoryCreate, DataResponse, CategoryResponse, CategoryPatch, DataPatch
from database import SessionLocal

router = APIRouter()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.put("/category/{category_id}", response_model=CategoryResponse)
async def update_category(category_id: int, category: CategoryCreate, db: Session = Depends(get_db)):
    db_category = db.query(Category).filter(category_id == Category.id).first()
    if db_category is None:
        raise HTTPException(status_code=404, detail="The category with the specified ID was not found")

    id_data = db.query(Data).filter(category.id == Category.id, category_id != Category.id).first()
    if id_data:
        raise HTTPException(status_code=400, detail="The category with this ID already exists")

    for key, value in category.dict().items():
        setattr(db_category, key, value)

    db.commit()
    db.refresh(db_category)
    return db_category

@router.get("/data/", response_model=List[DataResponse])
async def read_data(db: Session = Depends(get_db)):
    return db.query(Data).all()


@router.get("/data/{data_id}", response_model=DataResponse)
async def read_data_by_id(data_id: int, db: Session = Depends(get_db)):
    db_data = db.query(Data).filter(data_id == Data.id).first()
    if db_data is None:
        raise HTTPException(status_code=404, detail="The data with the specified ID was not found")
    return db_data


@router.post("/data/", response_model=DataResponse)
async def create_data(data: DataCreate, db: Session = Depends(get_db)):
    id_data = db.query(Data).filter(data.id == Data.id).first()
    if id_data:
        raise HTTPException(status_code=400, detail="The data with this ID already exists")

    name_data = db.query(Data).filter(data.name == Data.name).first()
    if name_data:
        raise HTTPException(status_code=400, detail="The data with this name already exists")

    id_category = db.query(Category).filter(data.category_id == Category.id).first()
    if not id_category:
        raise HTTPException(status_code=400, detail="The category with the specified ID was not found")

    db_data = Data(**data.dict())
    db.add(db_data)
    db.commit()
    db.refresh(db_data)

    return db_data


@router.put("/data/{data_id}", response_model=DataResponse)
async def update_data(data_id: int, data: DataCreate, db: Session = Depends(get_db)):
    db_data = db.query(Data).filter(data_id == Data.id).first()
    if db_data is None:
        raise HTTPException(status_code=404, detail=f"The data with the ID = {data_id} not found")

    id_data = db.query(Data).filter(data.id == Data.id, data_id != Data.id).first()
    if id_data:
        raise HTTPException(status_code=400, detail=f"The data with ID = {data_id} already exists")

    name_data = db.query(Data).filter(data.name == Data.name, data_id != Data.id).first()
    if name_data:
        raise HTTPException(status_code=400, detail="The data with this name already exists")

    if data.category_id:
        id_category = db.query(Category).filter(data.category_id == Category.id).first()
        if not id_category:
            raise HTTPException(status_code=400, detail=f"The category with ID = {category_id} not found")

    for key, value in data.dict().items():
        setattr(db_data, key, value)

    db.commit()
    db.refresh(db_data)
    return db_data


@router.delete("/data/{data_id}", response_model=DataResponse)
async def delete_data(data_id: int, db: Session = Depends(get_db)):
    db_data = db.query(Data).filter(data_id == Data.id).first()
    if db_data is None:
        raise HTTPException(status_code=404, detail=f"The data with the ID = {data_id} not found")

    db.delete(db_data)
    db.commit()
    return db_data


@router.patch("/data/{data_id}", response_model=DataResponse)
async def patch_data(data_id: int, data: DataPatch, db: Session = Depends(get_db)):
    db_data = db.query(Data).filter(data_id == Data.id).first()
    if db_data is None:
        raise HTTPException(status_code=404, detail=f"The data with the ID = {data_id} not found")

    updated_fields = data.dict(exclude_unset=True)

    if 'id' in updated_fields and updated_fields['id'] != data_id:
        id_data = db.query(Data).filter(Data.id == updated_fields['id']).first()
        if id_data:
            raise HTTPException(status_code=400, detail=f"The data with ID = {data_id} already exists")

    if 'name' in updated_fields:
        name_data = db.query(Data).filter(Data.name == updated_fields['name'], data_id != Data.id).first()
        if name_data:
            raise HTTPException(status_code=400, detail="The data with this name already exists")

    if 'category_id' in updated_fields:
        category = db.query(Category).filter(Category.id == updated_fields['category_id']).first()
        if not category:
            raise HTTPException(status_code=400, detail=f"The category with ID = {category_id} not found")

    for key, value in updated_fields.items():
        setattr(db_data, key, value)

    db.commit()
    db.refresh(db_data)
    return db_data


@router.get("/category/", response_model=List[CategoryResponse])
async def read_category(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    return db.query(Category).offset(skip).limit(limit).all()


@router.get("/category/{category_id}", response_model=CategoryResponse)
async def read_category_by_id(category_id: int, db: Session = Depends(get_db)):
    db_category = db.query(Category).filter(category_id == Category.id).first()
    if db_category is None:
        raise HTTPException(status_code=404, detail="The category with the specified ID was not found")
    return db_category


@router.post("/category/", response_model=CategoryResponse)
async def create_category(category: CategoryCreate, db: Session = Depends(get_db)):
    existing_category = db.query(Category).filter(category.id == Category.id).first()
    if existing_category:
        raise HTTPException(status_code=400, detail=f"The category with ID = {category_id} already exists")

    db_category = Category(**category.dict())
    db.add(db_category)
    db.commit()
    db.refresh(db_category)

    return db_category





@router.delete("/category/{category_id}", response_model=CategoryResponse)
async def delete_category(category_id: int, db: Session = Depends(get_db)):
    db_category = db.query(Category).filter(category_id == Category.id).first()
    if db_category is None:
        raise HTTPException(status_code=404, detail=f"The category with ID = {category_id} not found")

    related_data = db.query(Data).filter(category_id == Data.category_id).first()
    if related_data:
        raise HTTPException(status_code=403, detail="Cannot delete category with associated data")

    db.delete(db_category)
    db.commit()
    return db_category


@router.patch("/category/{category_id}", response_model=CategoryResponse)
async def patch_category(category_id: int, category: CategoryPatch, db: Session = Depends(get_db)):
    db_category = db.query(Category).filter(category_id == Category.id).first()
    if db_category is None:
        raise HTTPException(status_code=404, detail=f"The category with ID = {category_id} not found")

    updated_data = category.dict(exclude_unset=True)

    if 'id' in updated_data and updated_data['id'] != category_id:
        id_category = db.query(Category).filter(Category.id == updated_data['id']).first()
        if id_category:
            raise HTTPException(status_code=400, detail=f"The category with ID = {category_id} already exists")

    for key, value in updated_data.items():
        setattr(db_category, key, value)

    db.commit()
    db.refresh(db_category)
    return db_category
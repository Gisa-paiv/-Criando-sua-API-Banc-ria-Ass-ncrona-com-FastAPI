# -Criando-sua-API-Banc-ria-Ass-ncrona-com-FastAPI

#Estrutura final
src/
├── main.py
├── database.py
├── models.py
├── schemas.py
├── auth.py
├── dependencies.py
└── routes/
    ├── auth.py
    └── banking.py

from databases import Database
DATABASE_URL = "sqlite:///./bank.db"
database = Database(DATABASE_URL)   

import sqlalchemy

metadata = sqlalchemy.MetaData()

users = sqlalchemy.Table(
    "users",
    metadata,
    sqlalchemy.Column("id", sqlalchemy.Integer, primary_key=True),
    sqlalchemy.Column("username", sqlalchemy.String, unique=True),
    sqlalchemy.Column("password", sqlalchemy.String),
)

accounts = sqlalchemy.Table(
    "accounts",
    metadata,
    sqlalchemy.Column("id", sqlalchemy.Integer, primary_key=True),
    sqlalchemy.Column("balance", sqlalchemy.Float, default=0),
    sqlalchemy.Column("user_id", sqlalchemy.Integer),
)

transactions = sqlalchemy.Table(
    "transactions",
    metadata,
    sqlalchemy.Column("id", sqlalchemy.Integer, primary_key=True),
    sqlalchemy.Column("amount", sqlalchemy.Float),
    sqlalchemy.Column("type", sqlalchemy.String),
    sqlalchemy.Column("account_id", sqlalchemy.Integer),
)

from pydantic import BaseModel, Field

class LoginIn(BaseModel):
    username: str
    password: str

class TransactionIn(BaseModel):
    amount: float = Field(gt=0)  # não permite negativo
    type: str  # deposit / withdraw

from jose import jwt
from datetime import datetime, timedelta

SECRET_KEY = "secret"
ALGORITHM = "HS256"

def create_token(data: dict):
    data["exp"] = datetime.utcnow() + timedelta(hours=1)
    return jwt.encode(data, SECRET_KEY, algorithm=ALGORITHM)

from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

def authenticate_user(token=Depends(security)):
    if not token:
        raise HTTPException(status_code=401)
    return token

from fastapi import APIRouter
from src.schemas import LoginIn
from src.auth import create_token

router = APIRouter()

@router.post("/auth/token")
async def login(user: LoginIn):
    return {"access_token": create_token({"sub": user.username})}

from fastapi import APIRouter, Depends, HTTPException
from src.schemas import TransactionIn
from src.dependencies import authenticate_user

router = APIRouter()

fake_balance = 0
transactions = []

@router.post("/transactions")
async def create_transaction(
    data: TransactionIn,
    user=Depends(authenticate_user)
):
    global fake_balance

    # depósito
    if data.type == "deposit":
        fake_balance += data.amount

    # saque com validação
    elif data.type == "withdraw":
        if data.amount > fake_balance:
            raise HTTPException(
                status_code=400,
                detail="Saldo insuficiente"
            )
        fake_balance -= data.amount

    else:
        raise HTTPException(status_code=400, detail="Tipo inválido")

    transactions.append(data.dict())

    return {
        "message": "Transação realizada",
        "balance": fake_balance
    }


@router.get("/transactions")
async def get_statement(user=Depends(authenticate_user)):
    return {
        "balance": fake_balance,
        "transactions": transactions
    }    
    
from fastapi import FastAPI
from src.database import database
from src.routes import auth, banking

app = FastAPI(title="API Bancária Assíncrona")

app.include_router(auth.router)
app.include_router(banking.router)

@app.on_event("startup")
async def startup():
    await database.connect()

@app.on_event("shutdown")
async def shutdown():
    await database.disconnect()


uvicorn src.main:app --reload

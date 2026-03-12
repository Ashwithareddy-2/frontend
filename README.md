from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import openai
import pandas as pd
from sqlalchemy import create_engine, text
import os
from dotenv import load_dotenv

load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")

app = FastAPI()

# 1. Database Connection (Example using SQLite for demo, switch to Postgres for prod)
DATABASE_URL = "sqlite:///./sales_data.db"
engine = create_engine(DATABASE_URL)

# 2. Define the Database Schema (Context for the AI)
# The AI needs to know what tables exist to write valid SQL.
SCHEMA_CONTEXT = """
Table: sales
Columns: id (int), product_name (text), amount (float), sale_date (date)

Table: customers
Columns: id (int), name (text), email (text), region (text)
"""

class VoiceQuery(BaseModel):
    query: str

@app.post("/query")
async def process_voice_query(data: VoiceQuery):
    try:
        # 3. Generate SQL using LLM
        prompt = f"""
        You are a SQL expert. Convert the following natural language query into a valid SQL query.
        Use the provided schema context.
        
        Schema Context:
        {SCHEMA_CONTEXT}
        
        User Query: {data.query}
        
        Return ONLY the SQL query, no markdown formatting.
        """
        
        response = openai.ChatCompletion.create(
            model="gpt-4o", # or gpt-3.5-turbo
            messages=[{"role": "user", "content": prompt}]
        )
        
        generated_sql = response.choices[0].message.content.strip()

        # 4. Execute SQL
        with engine.connect() as conn:
            result = conn.execute(text(generated_sql))
            rows = result.fetchall()
            columns = result.keys()
            
            # Convert to JSON for frontend
            df = pd.DataFrame(rows, columns=[str(c) for c in columns])
            json_data = df.to_dict(orient='records')

        return {
            "sql": generated_sql,
            "data": json_data,
            "success": True
        }

    except Exception as e:
        return {"error": str(e), "success": False}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)# frontend

"""
Flask application for ecommerce sales insights dashboard.
Serves a web dashboard and provides API endpoints for sales data retrieval from SQLite.
"""

from flask import Flask, render_template, jsonify, request
from flask_cors import CORS
import sqlite3
import json
from datetime import datetime, timedelta
from functools import wraps
import os

app = Flask(__name__)
CORS(app)

# Configuration
DATABASE = 'sales_data.db'
app.config['JSON_SORT_KEYS'] = False


# ========================
# Database Helper Functions
# ========================

def get_db_connection():
    """Create a database connection with row factory for dict-like access."""
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn


def init_db():
    """Initialize the database with sample tables if they don't exist."""
    conn = get_db_connection()
    c = conn.cursor()
    
    # Create sales table
    c.execute('''
        CREATE TABLE IF NOT EXISTS sales (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            date TEXT NOT NULL,
            product_id INTEGER NOT NULL,
            product_name TEXT NOT NULL,
            category TEXT NOT NULL,
            quantity INTEGER NOT NULL,
            unit_price REAL NOT NULL,
            total_amount REAL NOT NULL,
            customer_id INTEGER NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Create products table
    c.execute('''
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            category TEXT NOT NULL,
            unit_price REAL NOT NULL,
            stock_quantity INTEGER NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Create customers table
    c.execute('''
        CREATE TABLE IF NOT EXISTS customers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            country TEXT,
            registration_date TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    conn.commit()
    conn.close()


def query_db(query, args=(), one=False):
    """Execute a database query and return results."""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute(query, args)
        rv = cur.fetchall()
        conn.close()
        return (rv[0] if rv else None) if one else rv
    except sqlite3.Error as e:
        return None


# ========================
# Error Handlers
# ========================

@app.errorhandler(400)
def bad_request(error):
    """Handle bad request errors."""
    return jsonify({'error': 'Bad request', 'message': str(error)}), 400


@app.errorhandler(404)
def not_found(error):
    """Handle not found errors."""
    return jsonify({'error': 'Not found', 'message': 'Resource not found'}), 404


@app.errorhandler(500)
def internal_error(error):
    """Handle internal server errors."""
    return jsonify({'error': 'Internal server error', 'message': str(error)}), 500


# ========================
# Route Decorators
# ========================

def validate_date_format(f):
    """Decorator to validate date format in query parameters."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        
        if start_date:
            try:
                datetime.strptime(start_date, '%Y-%m-%d')
            except ValueError:
                return jsonify({'error': 'Invalid start_date format. Use YYYY-MM-DD'}), 400
        
        if end_date:
            try:
                datetime.strptime(end_date, '%Y-%m-%d')
            except ValueError:
                return jsonify({'error': 'Invalid end_date format. Use YYYY-MM-DD'}), 400
        
        return f(*args, **kwargs)
    return decorated_function


# ========================
# Web Routes
# ========================

@app.route('/')
def dashboard():
    """Serve the main dashboard page."""
    return render_template('dashboard.html')


@app.route('/health')
def health_check():
    """Health check endpoint."""
    return jsonify({'status': 'healthy', 'timestamp': datetime.utcnow().isoformat()}), 200


# ========================
# API Routes - Sales Data
# ========================

@app.route('/api/sales', methods=['GET'])
@validate_date_format
def get_sales():
    """
    Retrieve sales data with optional filtering.
    
    Query Parameters:
    - start_date (YYYY-MM-DD): Filter sales from this date
    - end_date (YYYY-MM-DD): Filter sales until this date
    - category: Filter by product category
    - limit: Maximum number of records to return (default: 100)
    """
    try:
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        category = request.args.get('category')
        limit = request.args.get('limit', 100, type=int)
        
        query = 'SELECT * FROM sales WHERE 1=1'
        params = []
        
        if start_date:
            query += ' AND date >= ?'
            params.append(start_date)
        
        if end_date:
            query += ' AND date <= ?'
            params.append(end_date)
        
        if category:
            query += ' AND category = ?'
            params.append(category)
        
        query += ' ORDER BY date DESC LIMIT ?'
        params.append(limit)
        
        results = query_db(query, params)
        
        sales = [dict(row) for row in results]
        return jsonify({
            'success': True,
            'count': len(sales),
            'data': sales
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/sales/summary', methods=['GET'])
@validate_date_format
def get_sales_summary():
    """
    Get sales summary statistics.
    
    Query Parameters:
    - start_date (YYYY-MM-DD): Filter from this date
    - end_date (YYYY-MM-DD): Filter until this date
    """
    try:
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        
        query = 'SELECT SUM(total_amount) as total_sales, COUNT(*) as transaction_count, AVG(total_amount) as avg_transaction FROM sales WHERE 1=1'
        params = []
        
        if start_date:
            query += ' AND date >= ?'
            params.append(start_date)
        
        if end_date:
            query += ' AND date <= ?'
            params.append(end_date)
        
        result = query_db(query, params, one=True)
        
        if result:
            summary = {
                'total_sales': float(result['total_sales']) if result['total_sales'] else 0,
                'transaction_count': result['transaction_count'],
                'average_transaction': float(result['avg_transaction']) if result['avg_transaction'] else 0,
                'period': {
                    'start_date': start_date or 'all_time',
                    'end_date': end_date or 'all_time'
                }
            }
            return jsonify({'success': True, 'data': summary}), 200
        
        return jsonify({'success': False, 'error': 'No data found'}), 404
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/sales/daily', methods=['GET'])
@validate_date_format
def get_daily_sales():
    """
    Get daily sales aggregated data.
    
    Query Parameters:
    - start_date (YYYY-MM-DD): Filter from this date
    - end_date (YYYY-MM-DD): Filter until this date
    """
    try:
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        
        query = '''
            SELECT 
                date,
                SUM(total_amount) as total_sales,
                COUNT(*) as transaction_count,
                SUM(quantity) as total_quantity
            FROM sales
            WHERE 1=1
        '''
        params = []
        
        if start_date:
            query += ' AND date >= ?'
            params.append(start_date)
        
        if end_date:
            query += ' AND date <= ?'
            params.append(end_date)
        
        query += ' GROUP BY date ORDER BY date DESC'
        
        results = query_db(query, params)
        
        daily_sales = [{
            'date': row['date'],
            'total_sales': float(row['total_sales']),
            'transaction_count': row['transaction_count'],
            'total_quantity': row['total_quantity']
        } for row in results]
        
        return jsonify({
            'success': True,
            'count': len(daily_sales),
            'data': daily_sales
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/sales/category', methods=['GET'])
@validate_date_format
def get_sales_by_category():
    """
    Get sales data grouped by product category.
    
    Query Parameters:
    - start_date (YYYY-MM-DD): Filter from this date
    - end_date (YYYY-MM-DD): Filter until this date
    """
    try:
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        
        query = '''
            SELECT 
                category,
                COUNT(*) as transaction_count,
                SUM(total_amount) as total_sales,
                SUM(quantity) as total_quantity,
                AVG(total_amount) as avg_transaction
            FROM sales
            WHERE 1=1
        '''
        params = []
        
        if start_date:
            query += ' AND date >= ?'
            params.append(start_date)
        
        if end_date:
            query += ' AND date <= ?'
            params.append(end_date)
        
        query += ' GROUP BY category ORDER BY total_sales DESC'
        
        results = query_db(query, params)
        
        category_sales = [{
            'category': row['category'],
            'transaction_count': row['transaction_count'],
            'total_sales': float(row['total_sales']),
            'total_quantity': row['total_quantity'],
            'average_transaction': float(row['avg_transaction'])
        } for row in results]
        
        return jsonify({
            'success': True,
            'count': len(category_sales),
            'data': category_sales
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/sales/product', methods=['GET'])
@validate_date_format
def get_sales_by_product():
    """
    Get sales data grouped by product.
    
    Query Parameters:
    - start_date (YYYY-MM-DD): Filter from this date
    - end_date (YYYY-MM-DD): Filter until this date
    - limit: Maximum number of products (default: 50)
    """
    try:
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        limit = request.args.get('limit', 50, type=int)
        
        query = '''
            SELECT 
                product_id,
                product_name,
                category,
                COUNT(*) as transaction_count,
                SUM(total_amount) as total_sales,
                SUM(quantity) as total_quantity,
                AVG(unit_price) as avg_price
            FROM sales
            WHERE 1=1
        '''
        params = []
        
        if start_date:
            query += ' AND date >= ?'
            params.append(start_date)
        
        if end_date:
            query += ' AND date <= ?'
            params.append(end_date)
        
        query += ' GROUP BY product_id ORDER BY total_sales DESC LIMIT ?'
        params.append(limit)
        
        results = query_db(query, params)
        
        product_sales = [{
            'product_id': row['product_id'],
            'product_name': row['product_name'],
            'category': row['category'],
            'transaction_count': row['transaction_count'],
            'total_sales': float(row['total_sales']),
            'total_quantity': row['total_quantity'],
            'average_price': float(row['avg_price'])
        } for row in results]
        
        return jsonify({
            'success': True,
            'count': len(product_sales),
            'data': product_sales
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/sales/top-products', methods=['GET'])
def get_top_products():
    """
    Get top performing products by revenue.
    
    Query Parameters:
    - limit: Number of top products to return (default: 10)
    - start_date (YYYY-MM-DD): Filter from this date
    - end_date (YYYY-MM-DD): Filter until this date
    """
    try:
        limit = request.args.get('limit', 10, type=int)
        start_date = request.args.get('start_date')
        end_date = request.args.get('end_date')
        
        query = '''
            SELECT 
                product_name,
                SUM(total_amount) as revenue,
                SUM(quantity) as units_sold
            FROM sales
            WHERE 1=1
        '''
        params = []
        
        if start_date:
            query += ' AND date >= ?'
            params.append(start_date)
        
        if end_date:
            query += ' AND date <= ?'
            params.append(end_date)
        
        query += ' GROUP BY product_name ORDER BY revenue DESC LIMIT ?'
        params.append(limit)
        
        results = query_db(query, params)
        
        top_products = [{
            'product_name': row['product_name'],
            'revenue': float(row['revenue']),
            'units_sold': row['units_sold']
        } for row in results]
        
        return jsonify({
            'success': True,
            'count': len(top_products),
            'data': top_products
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


# ========================
# API Routes - Products
# ========================

@app.route('/api/products', methods=['GET'])
def get_products():
    """
    Retrieve all products.
    
    Query Parameters:
    - category: Filter by category
    - limit: Maximum number of records (default: 100)
    """
    try:
        category = request.args.get('category')
        limit = request.args.get('limit', 100, type=int)
        
        query = 'SELECT * FROM products WHERE 1=1'
        params = []
        
        if category:
            query += ' AND category = ?'
            params.append(category)
        
        query += ' LIMIT ?'
        params.append(limit)
        
        results = query_db(query, params)
        
        products = [dict(row) for row in results]
        return jsonify({
            'success': True,
            'count': len(products),
            'data': products
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/products/categories', methods=['GET'])
def get_categories():
    """Get all unique product categories."""
    try:
        results = query_db('SELECT DISTINCT category FROM products ORDER BY category')
        
        categories = [row['category'] for row in results]
        return jsonify({
            'success': True,
            'count': len(categories),
            'data': categories
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


# ========================
# API Routes - Customers
# ========================

@app.route('/api/customers', methods=['GET'])
def get_customers():
    """
    Retrieve customers.
    
    Query Parameters:
    - limit: Maximum number of records (default: 100)
    """
    try:
        limit = request.args.get('limit', 100, type=int)
        
        query = 'SELECT * FROM customers LIMIT ?'
        results = query_db(query, (limit,))
        
        customers = [dict(row) for row in results]
        return jsonify({
            'success': True,
            'count': len(customers),
            'data': customers
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


@app.route('/api/customers/count', methods=['GET'])
def get_customer_count():
    """Get total customer count."""
    try:
        result = query_db('SELECT COUNT(*) as count FROM customers', one=True)
        
        return jsonify({
            'success': True,
            'data': {'total_customers': result['count']}
        }), 200
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500


# ========================
# Main Entry Point
# ========================

if __name__ == '__main__':
    # Initialize the database
    init_db()
    
    # Run the Flask application
    app.run(
        host='0.0.0.0',
        port=5000,
        debug=True
    )

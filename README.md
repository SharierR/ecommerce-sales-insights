# E-Commerce Sales Insights Dashboard

A modern, interactive sales insights dashboard for e-commerce businesses built with Flask, SQLite, and Chart.js.

## Features

- 📊 Interactive sales charts and visualizations
- 📈 Real-time data analytics
- 💾 SQLite database for data persistence
- 🎨 Responsive, modern UI design
- 📱 Mobile-friendly dashboard
- 📥 CSV data import functionality

## Tech Stack

- **Backend**: Flask (Python)
- **Database**: SQLite
- **Frontend**: HTML5, CSS3, JavaScript
- **Charts**: Chart.js
- **Data Processing**: Pandas

## Project Structure

```
ecommerce-sales-insights/
├── app.py                          # Flask application
├── load_csv_to_sqlite.py          # CSV to SQLite data loader
├── requirements.txt                # Python dependencies
├── sales_data.csv                  # Sample sales data
├── sales.db                        # SQLite database (generated)
├── README.md                       # Project documentation
├── .gitignore                      # Git ignore rules
├── templates/
│   └── dashboard.html             # Main dashboard template
└── static/
    └── style.css                  # Dashboard styling
```

## Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/SharierR/ecommerce-sales-insights.git
   cd ecommerce-sales-insights
   ```

2. **Create a virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Load sample data**
   ```bash
   python load_csv_to_sqlite.py
   ```

5. **Run the application**
   ```bash
   python app.py
   ```

6. **Open your browser**
   Navigate to `http://localhost:5000` to view the dashboard

## Usage

### Loading Data

Place your CSV file in the project root directory with the following columns:
- `date` - Sales date
- `product` - Product name
- `quantity` - Quantity sold
- `price` - Unit price
- `total_sales` - Total sales amount
- `region` - Sales region

Then run:
```bash
python load_csv_to_sqlite.py
```

### Dashboard Features

- **Sales Overview**: Total sales, average order value, and key metrics
- **Sales Trends**: Monthly and daily sales visualization
- **Product Performance**: Top-selling products analysis
- **Regional Analysis**: Sales breakdown by region
- **Time-based Analytics**: Sales patterns and trends over time

## Database Schema

### sales table
- `id` (INTEGER, PRIMARY KEY)
- `date` (DATE)
- `product` (TEXT)
- `quantity` (INTEGER)
- `price` (REAL)
- `total_sales` (REAL)
- `region` (TEXT)

## Configuration

Edit `app.py` to customize:
- Flask port and debug mode
- Database location
- Chart refresh rates

## Future Enhancements

- [ ] User authentication and authorization
- [ ] Advanced filtering and date range selection
- [ ] Export reports to PDF/Excel
- [ ] Predictive analytics using ML models
- [ ] Real-time data updates with WebSockets
- [ ] Multi-user support with role-based access
- [ ] Dark mode theme

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Author

**SharierR** - Initial work and maintenance

## Support

For issues, questions, or suggestions, please open an issue on GitHub.

---

**Last Updated**: 2026-01-13
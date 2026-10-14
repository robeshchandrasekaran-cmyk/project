from flask import Flask, render_template, request, redirect, url_for, flash
import sqlite3

app = Flask(__name__)
app.secret_key = "change-this-before-deployment"
DB_NAME = "campus.db"


def connect_db():
    db = sqlite3.connect(DB_NAME)
    db.row_factory = sqlite3.Row
    return db


def initialize_db():
    with connect_db() as db:
        db.execute("""
            CREATE TABLE IF NOT EXISTS buses (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                bus_number TEXT NOT NULL,
                source TEXT NOT NULL,
                destination TEXT NOT NULL,
                departure TEXT NOT NULL,
                seats INTEGER NOT NULL
            )
        """)
        db.execute("""
            CREATE TABLE IF NOT EXISTS bookings (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                student_name TEXT NOT NULL,
                student_id TEXT NOT NULL,
                bus_id INTEGER NOT NULL,
                FOREIGN KEY (bus_id) REFERENCES buses(id)
            )
        """)

        count = db.execute(
            "SELECT COUNT(*) FROM buses"
        ).fetchone()[0]

        if count == 0:
            db.executemany("""
                INSERT INTO buses
                (bus_number, source, destination, departure, seats)
                VALUES (?, ?, ?, ?, ?)
            """, [
                ("CB101", "Main Gate", "Hostel", "08:00 AM", 40),
                ("CB102", "Hostel", "Library", "08:30 AM", 30),
                ("CB103", "Main Gate", "Engineering Block", "09:00 AM", 35),
                ("CB104", "Hostel", "Main Gate", "05:00 PM", 40),
            ])


@app.route("/")
def home():
    with connect_db() as db:
        buses = db.execute("""
            SELECT b.*,
                   b.seats - COUNT(k.id) AS available_seats
            FROM buses b
            LEFT JOIN bookings k ON b.id = k.bus_id
            GROUP BY b.id
            ORDER BY b.departure
        """).fetchall()

    return render_template("index.html", buses=buses)


@app.route("/book/<int:bus_id>", methods=["POST"])
def book_bus(bus_id):
    name = request.form.get("student_name", "").strip()
    student_id = request.form.get("student_id", "").strip()

    if not name or not student_id:
        flash("Please enter your name and student ID.")
        return redirect(url_for("home"))

    with connect_db() as db:
        bus = db.execute(
            "SELECT * FROM buses WHERE id = ?", (bus_id,)
        ).fetchone()

        if bus is None:
            flash("Bus not found.")
        else:
            booked = db.execute(
                "SELECT COUNT(*) FROM bookings WHERE bus_id = ?",
                (bus_id,)
            ).fetchone()[0]

            duplicate = db.execute("""
                SELECT id FROM bookings
                WHERE student_id = ? AND bus_id = ?
            """, (student_id, bus_id)).fetchone()

            if duplicate:
                flash("You have already booked this bus.")
            elif booked >= bus["seats"]:
                flash("Sorry, this bus is full.")
            else:
                db.execute("""
                    INSERT INTO bookings
                    (student_name, student_id, bus_id)
                    VALUES (?, ?, ?)
                """, (name, student_id, bus_id))
                flash("Seat booked successfully!")

    return redirect(url_for("home"))


@app.route("/my-bookings")
def my_bookings():
    student_id = request.args.get("student_id", "").strip()

    with connect_db() as db:
        bookings = db.execute("""
            SELECT k.id, k.student_name, k.student_id,
                   b.bus_number, b.source, b.destination,
                   b.departure
            FROM bookings k
            JOIN buses b ON b.id = k.bus_id
            WHERE k.student_id = ?
            ORDER BY k.id DESC
        """, (student_id,)).fetchall() if student_id else []

    return render_template(
        "index.html",
        buses=[],
        bookings=bookings,
        searched_id=student_id
    )


if __name__ == "__main__":
    initialize_db()
    app.run(debug=True)

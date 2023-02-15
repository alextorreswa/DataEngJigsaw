from api.models import Assignment, Depot, Route, Vehicle, VehicleSpecs, Make, Model, BodyType, BodySubtype, DriveType, Fuel, Swap, User
from api.services import SwapCalculation, SwapSheet, SwapBuilder, SwapService, MatchVehicleBuilder, MatchVehicle, FleetPost
from flask import Flask, render_template, request, session, g, redirect, url_for
from api.lib.db import conn, cursor, re_connect
from api.auth import login_required
from api.lib.orm import build_from_record
import os


def create_app(test_config=None):
    # create and configure the app
    app = Flask(__name__, instance_relative_config=True)
    app.config.from_mapping(
        SECRET_KEY='dev'
    )

    # configure mail client
    app.config['MAIL_SERVER'] = 'smtp.gmail.com'
    app.config['MAIL_PORT'] = 465
    app.config['MAIL_USERNAME'] = 'swapbusapp@gmail.com'
    app.config['MAIL_PASSWORD'] = 'eoruyxeetsmkestq'
    app.config['MAIL_USE_TLS'] = False
    app.config['MAIL_USE_SSL'] = True

    if test_config is None:
        # load the instance config, if it exists, when not testing
        app.config.from_pyfile('config.py', silent=True)
    else:
        # load the test config if passed in
        app.config.from_mapping(test_config)

    @login_required
    @app.route('/', methods=['GET', 'POST'])
    def index():
        try:
            # require an authenticated user to login
            if g.user is None:
                return redirect(url_for('auth.login'))

            global options, bus_to_swap

            # evaluate POST request
            if request.method == 'POST':

                # find options by route number
                if request.form.get('option') == 'route':
                    # before find by route validate route numbers ex: K559 - no numerics, just four characters
                    if not request.form.get('number').isnumeric() and len(request.form.get('number')) == 4:
                        # use upper to convert any number to capital letters and find any case
                        bus_by_route = Vehicle.find_id_by_route_flask_app(request.form.get('number').upper(), cursor)
                        if bus_by_route is not None:
                            # determine if the search is intradepot or interdepot
                            mode_depot = 'intradepot' if request.form.get('intradepot') else 'interdepot'
                            # create instance of SwapService with the buses available to swap
                            swap_service = SwapService(bus_by_route.nycsbus_id, mode_depot)
                        else:
                            # display wrong number
                            return render_template("swap/options.html", number=request.form.get('number'),
                                                   wrong_number=True,
                                                   option=request.form.get('option'), no_options=False, no_bus=True)
                    else:
                        # display wrong number
                        return render_template("swap/options.html", number=request.form.get('number'), wrong_number=True,
                                               option=request.form.get('option'), no_options=False, no_bus=True)

                # bus option by bus number
                elif request.form.get('option') == 'bus':
                    # validate just numeric numbers
                    if request.form.get('number').isnumeric():
                        # determine if the search is intradepot or interdepot
                        mode_depot = 'intradepot' if request.form.get('intradepot') else 'interdepot'
                        # create instance of SwapService with the buses available to swap
                        swap_service = SwapService(request.form.get('number'), mode_depot)
                    else:
                        # display wrong number
                        return render_template("swap/options.html", number=request.form.get('number'), wrong_number=True,
                                               option=request.form.get('option'), no_options=False, no_bus=True)

                # show options if the user put a valid number and an option
                if request.form.get('option') and swap_service:
                    # load the bus to swap
                    bus_to_swap = swap_service.find_vehicle(cursor)
                    if bus_to_swap:
                        # load vehicles available (list of objects) for the bus to swap
                        options = swap_service.run(cursor)
                        if options:
                            # display options available
                            return render_template("swap/options.html", number=request.form.get('number'),
                                                   options=[o.__dict__ for o in options], wrong_number=False,
                                                   option=request.form.get('option'), no_options=False,
                                                   bus=bus_to_swap.description(conn))
                        else:
                            # display no options available
                            return render_template("swap/options.html", no_options=True, hidden=False,
                                                   option=request.form.get('option'), number=request.form.get('number'))
                    else:
                        # display wrong number
                        return render_template("swap/options.html", number=request.form.get('number'), wrong_number=True,
                                               option=request.form.get('option'), no_options=False, no_bus=True)

                # make the swap if the user selected an option and push the button make a swap
                if bool(request.form.get('swap_option')) and bus_to_swap:
                    # create a user object
                    user = build_from_record(User, g.user)
                    # bus object that the user chose
                    chosen_bus = [o for o in options if o.nycsbus_id == int(request.form.get("swap_option"))][0]
                    # make the swap
                    # update database and google sheet
                    swap_response = SwapBuilder(bus_to_swap, chosen_bus, user, request.form.get("swap_reason")).run(conn, cursor)
                    # POST request to Fleet.io to update availability status of vehicle being replaced
                    FleetPost().run(bus_to_swap.fleetio_id)
                    # display successful message to the user
                    return render_template("swap/success.html", bus_to_swap=bus_to_swap.nycsbus_id,
                                           option=request.form.get("swap_option"), reason=request.form.get("swap_reason"),
                                           user_id=session.get("user_id"))

                    # f'swap {bus_to_swap.nycsbus_id} with {request.form.get("swap_option")} reason {request.form.get("swap_reason")} made by {session.get("user_id")}'

            return render_template("swap/index.html")
        except Exception as e:
            # try to reconnect the application if the server was down, uncomment/comment the next two lines if you want to use and avoid display debug messages to users.
            # re_connect()
            # return redirect(url_for('auth.login'))
            return render_template(str(e))

    # authentication
    from . import auth
    app.register_blueprint(auth.bp)
    return app


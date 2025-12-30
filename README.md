# DEV2IL: Observability

## The Smoothie Shop

The smoothie shop application, allows users to order delicious smoothies. It consists of two microservices:
- The Order Service: Accepts smoothie orders
- The Kitchen Service: Prepares the smoothies

To open your personal smoothie shop
- Open a terminal and run `uv run uvicorn order_service:app --port 8000 --reload`. 
- Open another terminal and run: `uv run uvicorn kitchen_service:app --port 8001 --reload`.

## Operating the Smoothie Shop in Blind Mode

Let's start to buy some smoothies. Open a terminal and run `uv run buy_smoothies.py`. 
Look at the console output. You should see that your smoothie shop is working fine.

Let's start to send some more customers to your smoothie shop. Open another terminal and run 
`uv run buy_smoothies.py`. Look at the console output again.

It looks like your shop is having some troubles from time to time. Try to figure out what is going wrong by
looking at the outputs of all the started services. **You are not allowed to look at the code!** 
Could you figure it out and fix it ?

Most likely, you've been unable to tell why the application failed from time to time. The only way to 
find out is to ask the developers. If you look into the code of `kitchen_service.py`, you will notice
that the kitchen rejects a request to prepare a smoothie with a status code of 503 if all cooks are
so busy that the work on the requested smoothie can't be started in time. In this case, the fix would 
have been easy, as the kitchen already contains a configuration parameter to increase the number of cooks
(`NUM_COOKS`).

## Providing More Insights Through Log Outputs

We are now providing more insights into the smoothie shop by adding logging to the application. Remember 
these hints on which logging level to choose from the [Python logging HOWTO](https://docs.python.org/3/howto/logging.html#when-to-use-logging): 

| Level     | When it’s used                                                                                       |
|-----------|------------------------------------------------------------------------------------------------------|
| DEBUG     | Detailed information, typically of interest only when diagnosing problems.                           |
| INFO      | Confirmation that things are working as expected.                                                    |
| WARNING   | An indication that something unexpected happened, or indicative of some problem in the near future. (e.g. ‘disk space low’). The software is still working as expected. |
| ERROR     | Due to a more serious problem, the software has not been able to perform some function.              |
| CRITICAL  | A serious error, indicating that the program itself may be unable to continue running.               |


Modify `kitchen_service.py`

- After the existing imports create a logger for the module
```python
import logging
logger = logging.getLogger(__name__)
``` 
- Add these log messages to the `prepare_smoothie` function. Find the right places to add them on your own. 
```python
logger.info(f"Received order to prepare a smoothie with flavor {order.flavor}")
logger.debug(f"Waiting for a cook to become available")
logger.error(f"Can't process the order: {NUM_COOKS} cooks are currently busy. Consider increasing NUM_COOKS.")
logger.info(f"Smoothie with flavor {order.flavor} prepared")
```

We want all our logging messages to contain the logging level, a timestamp when the message was logged and 
the message itself. In addition, we want to be able to define the logging level for each logger individually.
Download the file [logging_config.yaml](https://github.com/peterrietzler/ais-dev2il-smoothie-shop/blob/logging/logging_config.yaml)
and store it in the root directory of the project.

Stop the kitchen service and start it again using 
`uv run uvicorn kitchen_service:app --port 8001 --reload --log-config logging_config.yaml`.  

You can now adjust the log levels, by setting the level of detail that you want to see in `logging_config.yaml`.

### Further Readings and Exercises

TODOs -

merge into master

- Read through the [Python Logging HOWTO](https://docs.python.org/3/howto/logging.html)
- https://docs.python.org/3/library/logging.html#
- Introduce proper logging in the order service
- Correlate log messages

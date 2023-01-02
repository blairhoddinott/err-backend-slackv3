## Overview

This is a branch of the [official](https://github.com/errbotio/err-backend-slackv3) err-backend-slackv3 repo. The original repo was missing functionality for Slack interactive messaging as of January 1, 2023. I have squeezed this functionality in, in not necessarily the cleanest fashion.

## What Was Changed

`src/slackv3/slackv3.py`

There were two modifications to the main library. One was in the `_generic_wrapper` function:

```python
def _generic_wrapper(self, event_data):
        """Calls the event handler based on the event type"""
        log.debug("Received event: {}".format(str(event_data)))
        if event_data['type'] == "block_actions" or \
           event_data['type'] == "interactive_message":
            return self._interactive_message_event_handler(event_data)
        try:
            event = event_data["event"]
            event_type = event["type"]
...
```

Specifically, this was added:

```python
if event_data['type'] == "block_actions" or \
           event_data['type'] == "interactive_message":
            return self._interactive_message_event_handler(event_data)
```

This checks to see if there is a `block_actions` or `interactive_message` message coming back through the websocket. If so, we hand off the event to a new function `_interactive_messsge_event_hander`.

The next change was the addition of the `_interactive_message_event_handler` function:

```python
def _interactive_message_event_handler(self, event):
        """
        Event handler for the 'interactive' event, used in attachments / buttons.
        """
        log.debug(f"event: {event}")
        msg = Message(
            frm=SlackPerson(self.slack_web, event['user']['id'], event['channel']['id']),
            to=self.bot_identifier,
            extras={
                'actions': event['actions'],
                'url': event['response_url'],
                'trigger_id': event.get('trigger_id', None),
                'callback_id': event.get('callback_id', None),
                'ts': event.get('message_ts', None),
                'channel': event['channel']['id'],
                'user': event['user']['id'],
                'slack_event': event
            }
        )

        flow, _ = self.flow_executor.check_inflight_flow_triggered(msg.extras['callback_id'], msg.frm)
        if flow:
            log.debug("Reattach context from flow %s to the message", flow._root.name)
            msg.ctx = flow.ctx

        self.callback_message(msg)
```

This will grab the event, pull some info out which will be convenient to grab and pass it along as an `extras` dictionary to the msg parameter when we initiate a callback. 

This code was mainly pulled from [here](https://github.com/errbotio/err-backend-slackv3/issues/40#issuecomment-742750196). **MANY** thanks to [@duhow](https://github.com/duhow) for his code. 

## Example Usage

Install this slack backend instead of the original backend to the robot. 

In your plugin, you will need the following:
- Callback function
- Callback handler function

### Callback function

In the main class of your plugin, add this:

```python
def callback_message(self, msg):
        log.debug("Callback initiated...")
        callback = msg.extras.get('callback_id', None)
        log.debug(f"callback: {callback}")
        if (
            callback and callback in dir(self) and
            callable(getattr(self, callback))
        ):
            self.log.info(f'Calling function {callback}')
            return getattr(self, callback)(msg)
```

This will receive the callback from the slack library, and look for the callback in your code. If it finds it, it will then initiate the callback.

### Callback handler

In this example, when I initially sent the slack message, I used a `callback_id` of `approval_request`. When the callback handler above reads the callback_id, it will look for a botcmd with that function. 

```python
@botcmd
    def approval_request(self, msg, args=None) -> None:
        log.debug(f"msg: {msg.extras.get('callback_id', None)}")
        if msg.extras.get('callback_id', None) == 'approval_request':
            answer = msg.extras.get('actions')[0]['value']
            ts = msg.extras.get('ts', None)
            channel = msg.extras.get('channel', None)
            log.debug(f"channel: {channel}")

            ...
            <logic to modify original slack message>
            ...

            if ts:
                SLACK.chat_update(
                    channel=channel,
                    text=answer.title(),
                    blocks=blocks,
                    attachments=attachments,
                    ts=ts
                )
```

In this example, I verify the callback, grab the important info from the extras we spliced off in the backend libarary. I then modify my blocks as required, and run a `chat_update` to update my message as desired.


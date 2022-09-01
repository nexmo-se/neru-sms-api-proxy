# neru-sms-api-proxy

Store client_ref inside the DLR endpoint and schedule to delete it.

```js
const state = neru.getGlobalState();
const scheduler = new Scheduler(neru.createSession());
const reminderTime = new Date(
  // DELETE AFTER 24 HOURS
  new Date().setHours(new Date().getHours() + 24)
  // FOR TESTING: delete client_ref after 1 minute
  // new Date().setMinutes(new Date().getMinutes() + 1)
).toISOString();
// USE MSISDN AS KEY
const payloadKey = `${req.query.msisdn}`;
console.log('scheduled setup', reminderTime);
// EXECUTE CLEANUP ENDPOINT TO DELETE SAVED CLIENT_REF
scheduler
  .startAt({
    startAt: reminderTime,
    callback: 'cleanup',
    payload: {
      key: payloadKey,
    },
  })
  .execute()
  .then(() => {
    console.log('session saved!');
  });
```

Cleanup Endpoint - helper to delete after 24 hours

```js
app.post('/cleanup', async (req, res) => {
  const state = neru.getGlobalState();
  console.log('cleanup job executed', req.body);
  await state.del(req.body.key);
  res.status(200).send('OK');
});
```

On Inbound webhook endpoint, see if stored client_ref exists using `msisdn` as key.
If found (not expired in 24 hours) return it, if not found send without it.

```js
const state = neru.getGlobalState();
const foundEntry = await state.get(`${req.query.msisdn}`);
console.log('record retrieved:', foundEntry, req.query.msisdn);
```

Neru server with 2 endpoints (`DLR` and `Inbound`) to store `client_ref` from DLR to database, then looks up the client_ref on Inbound.
A third endpoint `cleanup` is used as part of a callback for scheduled deletion of the client_ref.

NERU SMS API Endpoints:
DLR:
https://NERU-DEPLOY-URL/webhooks/delivery-receipt
INBOUND:
https://NERU-DEPLOY-URL/webhooks/inbound

2 Requests are made to allow another server to recieve message payload. Replace the Server URL with your own. E.g.

```js
// from-dlr
'http://kittphi.ngrok.io/from-dlr';
// from-inbound
'http://kittphi.ngrok.io/from-inbound';
```

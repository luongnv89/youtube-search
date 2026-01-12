# Render Deployment Troubleshooting Guide

Having issues with your Render deployment? This guide covers common problems and solutions.

## Deployment Issues

### Build Failed

**Error**: "Build failed" shown on Render dashboard

**Possible Causes**:
- Missing or broken `package.json`
- Node.js version incompatibility
- Missing dependencies

**Solutions**:

1. **Check Logs**
   - Go to Render dashboard
   - Click your service
   - Open **"Logs"** tab
   - Look for error messages

2. **Verify Files**
   ```bash
   # Make sure these files exist in your repository
   ls proxy-server.js
   ls package.json
   ls render.yaml
   ```

3. **Update Node Version** (in render.yaml):
   ```yaml
   services:
     - type: web
       name: youtube-proxy
       env: node
       buildCommand: npm install
       startCommand: node proxy-server.js
       envVars:
         - key: NODE_ENV
           value: production
   ```

4. **Try npm ci instead**:
   - Go to settings → Environment
   - Change buildCommand to `npm ci`
   - Redeploy

### Deploy Never Completes

**Error**: "Deploying..." status never changes

**Solutions**:

1. **Check if service is starting**:
   - Wait 5 minutes for initial deploy
   - If still deploying, click **"Manual Deploy"** → **"Trigger Deploy"** again

2. **Increase timeout** (rare):
   - Try canceling and redeploying
   - Go to **"Settings"** → **"Delete"** → redeploy

3. **Check disk space**:
   - Logs show `ENOSPC: no space left on device`
   - Solution: Upgrade to paid plan or clear old builds

## Runtime Issues

### Service Crashes or Keeps Restarting

**Symptoms**: Service shows "Crashed", status keeps changing

**Check Logs**:
```
Error: listen EADDRINUSE: address already in use :::3000
```

**Solutions**:

1. **Fix PORT Environment Variable**
   - Go to **"Settings"** → **"Environment"**
   - Make sure `PORT=10000` is set
   - Not `PORT=3000` or blank
   - Click **"Save"** and **"Restart Service"**

2. **Check for Syntax Errors**
   ```bash
   # Run locally to verify
   node proxy-server.js
   ```
   Fix any errors shown

3. **Clear Build Cache**
   - Click three-dot menu
   - Select **"Rebuild"**
   - Wait for redeploy

### Service Running but Returns Errors

**Error**: Service is online but proxied requests fail

**Check Logs**:
1. Make a request: `curl https://youtube-proxy.onrender.com/proxy?url=...`
2. Check logs in Render dashboard
3. Look for error messages

**Common Errors**:

| Error | Cause | Fix |
|-------|-------|-----|
| `ECONNREFUSED` | Can't reach YouTube | YouTube servers down or blocked |
| `404 Not Found` | Wrong URL format | Check proxy URL syntax |
| `403 Forbidden` | URL not whitelisted | Only youtube.com domains allowed |
| `500 Internal Server` | Server error | Check logs, restart service |

## Connectivity Issues

### Cannot Connect to Proxy

**Symptoms**:
```
Error: getaddrinfo ENOTFOUND youtube-proxy.onrender.com
```

**Solutions**:

1. **Verify URL is correct**
   - Check service name on Render dashboard
   - Format: `https://[SERVICE-NAME].onrender.com/proxy?url=`
   - Example: `https://youtube-proxy.onrender.com/proxy?url=`

2. **Service might be asleep**
   - Free tier services sleep after 15 minutes of inactivity
   - First request takes 30-60 seconds to wake up
   - Solution: Upgrade to Starter plan to disable sleep

3. **DNS propagation**
   - Wait 5-10 minutes for DNS to update
   - Try in incognito mode or different browser

4. **Network blocked**
   - Check if firewall blocks outgoing requests
   - Try from different network
   - Check corporate VPN restrictions

### Timeout Errors

**Error**: Request times out without response

**Causes**:
- Render service asleep (free tier)
- YouTube server slow or unresponsive
- Network connectivity issue

**Solutions**:

1. **Wait for wake-up** (free tier):
   ```javascript
   // Add timeout retry logic
   const fetchWithRetry = async (url, retries = 3) => {
     for (let i = 0; i < retries; i++) {
       try {
         return await fetch(url, { timeout: 10000 });
       } catch (error) {
         if (i === retries - 1) throw error;
         await new Promise(r => setTimeout(r, 2000));
       }
     }
   };
   ```

2. **Upgrade to Starter plan**:
   - Eliminates cold starts
   - $7/month
   - Better performance

## YouTube Blocking

### No Results from YouTube

**Symptoms**:
- Proxy returns success (200 OK)
- But search returns 0 results
- Or returns error page HTML

**Causes**:
- YouTube detecting automated requests
- IP address rate-limited
- YouTube changed API structure

**Solutions**:

1. **Wait and Retry**
   - YouTube rate-limits aggressive requests
   - Wait 10-30 minutes
   - Try again later

2. **Rotate IPs** (Paid plan)
   - Deploy new instance on paid plan
   - YouTube might not recognize new IP
   - Old free tier IP might be rate-limited

3. **Use VPN/Proxy Chain**
   - Deploy proxy on different hosting (AWS, GCP)
   - Your proxy → VPN → YouTube

4. **Check if YouTube API Changed**
   - Look at YouTube HTML structure
   - Verify parser.js still works
   - May need to update search logic

5. **Contact YouTube**
   - If you need persistent access
   - Apply for official YouTube Data API key
   - More reliable long-term

## Performance Issues

### Slow Response Times

**Symptoms**: Requests taking 5+ seconds

**Possible Causes**:
- Cold start (free tier)
- YouTube server slow
- Network latency

**Solutions**:

1. **First request after sleep** (free tier only):
   - Expected: 30-60 second delay
   - Upgrade to Starter plan to avoid

2. **Check Render metrics**:
   - Go to **"Metrics"** tab
   - Check CPU and memory usage
   - If maxed out, upgrade plan

3. **YouTube servers slow**:
   - Try at different time
   - Check YouTube status at [status.youtube.com](https://status.youtube.com)

4. **Network optimization**:
   ```javascript
   // Add request timeout
   const client = new YouTubeClient({
     proxyUrl: 'https://youtube-proxy.onrender.com/proxy?url=',
     timeout: 10000  // 10 second timeout
   });
   ```

### High Bandwidth Usage

**Symptoms**: Warning about exceeding free tier bandwidth

**Solutions**:

1. **Check if proxy is being abused**
   - Review Render logs
   - Look for spam/unusual requests
   - Block abusive IPs if needed

2. **Optimize requests**:
   - Use result `limit` parameter
   - Cache results with `useCache: true`
   - Reduce search frequency

3. **Upgrade plan**:
   - Free tier: 100GB/month bandwidth
   - Starter plan: Unlimited

## Configuration Issues

### Wrong Service Name on Render

**Problem**: Service created but name not as expected

**Solution**:
- Render automatically names services
- If you want to rename:
  1. Go to service **"Settings"**
  2. Click **"Change Name"**
  3. Enter new name
  4. New URL: `https://new-name.onrender.com`

### Port Number Wrong

**Problem**: Service shows different port than expected

**Check**:
- Render assigns port 10000 on free tier
- This is set in render.yaml: `PORT: 10000`
- You don't connect directly to port 10000
- Use the public URL: `https://youtube-proxy.onrender.com`

### Environment Variables Not Working

**Problem**: Environment variables not being used

**Solutions**:

1. **Verify variables set**:
   - Go to **"Settings"** → **"Environment"**
   - Check all variables listed
   - Look for typos

2. **Restart service**:
   - After changing environment variables
   - Click menu → **"Restart Service"**
   - Old values cached until restart

3. **Check in code**:
   ```javascript
   // Verify variable is read correctly
   console.log('PORT:', process.env.PORT);
   // Should print: PORT: 10000
   ```

## Database/Persistence Issues

### Logs Not Persisting

**Problem**: Historical logs disappear

**Note**: Render keeps logs for 30 days only

**Workaround**: Save logs locally for auditing:
```bash
# Periodic log export script
curl https://render.com/api/v1/services/[service-id]/logs \
  -H "Authorization: Bearer $RENDER_API_KEY" > logs-backup.txt
```

## Getting Help

### Debug Mode

Enable debug logging:

1. In render.yaml, add:
   ```yaml
   envVars:
     - key: DEBUG
       value: 'true'
   ```

2. Check logs for detailed output

### Check Render Status

Sometimes Render has outages:
- [status.render.com](https://status.render.com)
- Check if all systems are operational

### Report an Issue

If you found a bug in the proxy:

1. Test locally first:
   ```bash
   node proxy-server.js
   ```

2. If it works locally but not on Render:
   - Check environment variables
   - Look for PORT issues
   - Review logs carefully

3. Report issue on GitHub:
   - Include logs from Render
   - Include exact error message
   - Include steps to reproduce

## Quick Reference

### Common Commands

**Restart service**:
- Render dashboard → service → menu → "Restart Service"

**View logs**:
- Render dashboard → service → "Logs" tab

**Rebuild service**:
- Render dashboard → service → menu → "Rebuild"

**Delete service** (and redeploy):
- Render dashboard → service → "Settings" → "Delete Service"

### Important URLs

- Render Dashboard: https://dashboard.render.com
- Render Status: https://status.render.com
- Render Docs: https://render.com/docs

### Environment Variables Required

```
PORT=10000
NODE_ENV=production
```

## Still Having Issues?

1. **Read detailed guide**: [RENDER_DEPLOYMENT.md](./RENDER_DEPLOYMENT.md)
2. **Check YouTube Search repo**: GitHub issues section
3. **Render support**: https://render.com/support
4. **Test locally first**: `node proxy-server.js`

---

**Pro Tip**: Always test changes locally before pushing to Render!

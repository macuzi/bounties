# Unable to connect the postgres db to CF Hyperdrive

### Deploy PostgreSQL Database on Railway

```bash
# Login
railway login

# Create new project
railway init

# Add PostgreSQL database
railway add --database postgres

```

### Get Database Connection Details

Go to the Variables tab and Copy the `DATABASE_URL`:

<img src="https://files.readme.io/196eada2628123a35c5f13284ae2052bf308ae9f72f7d4a5b625b73441958eca-Screenshot_2025-10-18_at_11.15.20_PM.png" />

### Verify Database SSL Configuration

```bash
# Test connection with SSL
psql "postgresql://postgres:password@hostname.railway.app:5432/railway?sslmode=require"

# here is a nice blog for psql commands https://www.tigerdata.com/blog/how-to-test-your-postgresql-connection
```

My response below (I’m two versions behind the server as told by `server major version 17`, but I should still be able to run standard operations needed for this): 

<img src="https://files.readme.io/f80a2351ed3c9815fa1eb2461c92fdfbb3bcdfeddf9a2400daf6ec0c35c9e7b0-Screenshot_2025-10-18_at_11.47.48_PM.png" />

### Check Initial Certificate Expiry

```bash
# Replace hostname.railway.app and 5432 with your actual values
openssl s_client -starttls postgres -connect hostname.railway.app:5432 | openssl x509 -noout -dates
```

<img src="https://files.readme.io/5dd668bf9b2c3d53cede80a530a512633b0e3f72457de26b575ba6506229047d-Screenshot_2025-10-18_at_11.50.22_PM.png" />

**Save this output** - you'll compare it after regeneration.


- So this means
    
    Time until expiration: ~820 days (about 2 years and 3 months)
    
    This is a fresh, valid certificate - my Railway PostgreSQL was just deployed recently, so it has brand new certificates.
    

*It’s easy to forget what each flag does (I know I do), so here’s a quick reference::* 

- `openssl s_client`
    
    OpenSSL's SSL/TLS client tool - like a browser that connects to SSL/TLS servers to inspect certificates.
    
- `starttls postgres`
    
    What it means:
    
    - PostgreSQL doesn't start with SSL immediately.
    - The client connects normally first, then upgrades to SSL/TLS using the STARTTLS protocol.
    - `starttls postgres` tells OpenSSL to use PostgreSQL's specific STARTTLS handshake.
    
- `-connect [turntable.proxy.rlwy.net:30765](http://turntable.proxy.rlwy.net:30765/)`
    
    Specifies where to connect (hostname:port)
    
- `|`
    
    What it does: Takes the output from the first command and feeds it as input to the next command
    
- `openssl x509 -noout -dates`
    
    What it does: Certificate parsing tool
    
    - x509 = the certificate format standard
    - noout = don't output the entire certificate (too much info)
    - dates = only show the validity dates (notBefore, notAfter)

[openssl-s_client - OpenSSL Documentation](https://docs.openssl.org/3.0/man1/openssl-s_client/)

[openssl-x509 - OpenSSL Documentation](https://docs.openssl.org/3.0/man1/openssl-x509/)

### Set Up Cloudflare Workers

```bash
# Login to Cloudflare
wrangler login
```

### Create Initial Hyperdrive Configuration

```bash
# Replace with your Railway DATABASE_URL
npx wrangler hyperdrive create railway-test \\
  --connection-string="postgresql://postgres:password@hostname.railway.app:5432/railway?sslmode=require"

# Expected output:
{
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<random id from HYPERDRIVE>"
    }
  ]
}
```

My example below: 

<img src="https://files.readme.io/f1fc7121c258c10a9e04f415ca2a5a6056a45f1da9e63807f0030b6d522e075b-Screenshot_2025-10-18_at_11.59.37_PM.png" />

Link on hyperdrive create: 

[Wrangler commands](https://developers.cloudflare.com/hyperdrive/reference/wrangler-commands/#hyperdrive-create)

### Now I believe we have things in place to test the fix. We’ll do a baseline test with a fresh database

This verifies that new databases work correctly (which was one of your observations).

```bash
# Your fresh Railway PostgreSQL should 
#connect to Hyperdrive without issues

npx wrangler hyperdrive get <id>
```

<img src="https://files.readme.io/172a1e642c91ca0a7cae307f1a105fc1b88449b32e2e1b5605acbf6ad928fd95-Screenshot_2025-10-19_at_12.03.01_AM.png" />

### Simulate Certificate Issue

Since we can't easily expire certificates, we'll test the regeneration process instead.

### Delete this REGENERATE_CERTS (or set to false)

<img src="https://files.readme.io/6fb8f228b1f9128b59f72c5fd0593e220eb887b13dfa371ea1c75e8f52125c84-Screenshot_2025-10-19_at_12.07.05_AM.png" />

```bash
openssl s_client -starttls postgres -connect hostname.railway.app:5432 | openssl x509 -noout -dates

# notBefore should be recent (within last few minutes)
# notAfter should be ~820 days in future
```

### Test Hyperdrive Connection After Regeneration

```bash
# Delete old Hyperdrive config
npx wrangler hyperdrive delete railway-test

# Create new config with fresh certificates
npx wrangler hyperdrive create railway-test \\
  --connection-string="postgresql://postgres:password@hostname.railway.app:5432/railway?sslmode=require"

# Verify connection
npx wrangler hyperdrive get <id>
```

## Verification Checklist

After completing the tests, verify:

- [ ]  Fresh Railway PostgreSQL database has valid SSL certificates (820-day expiry)
- [ ]  `REGENERATE_CERTS=true` successfully regenerates certificates
- [ ]  New certificate dates show recent `notBefore` after regeneration
- [ ]  Hyperdrive connects successfully with `sslmode=require`
- [ ]  Deleting and recreating Hyperdrive config works after certificate regeneration
- [ ]  Direct `psql` connection shows SSL connection info

## Alternative Approach

If the `REGENERATE_CERTS` doesn't work, you could also just try redeploying your PostgreSQL service from the Railway dashboard. Railway's setup script checks certificate expiry on startup and should auto-regenerate if they're expired or about to expire (within 30 days).

## Why This Happened

Old databases that haven't been redeployed in a long time may have outdated SSL certificates. Railway's PostgreSQL certificates typically expire after around 820 days. If your database was created a couple of years ago and hasn't been redeployed, the certificates might have just hit their expiration date.

## Where I Got This Info

I found the `REGENERATE_CERTS` solution from Railway's Help Station where their team recommended it for similar certificate issues: https://station.railway.com/questions/issue-with-postgres-ssl-certificate-date-b9c8cde4

The Railway postgres-ssl image that handles all this is here: https://github.com/railwayapp-templates/postgres-ssl

Good luck! 

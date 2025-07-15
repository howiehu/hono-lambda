# Hono-Lambda Project Guide

A serverless web application using Hono framework with AWS Lambda, managed by AWS CDK and developed locally with SAM CLI.

## Overview

This project demonstrates the AWS-recommended approach for serverless development:
- **Local Development**: SAM CLI with API Gateway for testing and debugging
- **Production Deployment**: AWS CDK with Lambda Function URL for efficiency

## Technology Stack

- **Framework**: [Hono](https://hono.dev/) - Fast, lightweight web framework
- **Runtime**: Node.js 22.x
- **Infrastructure as Code**: AWS CDK (Cloud Development Kit)
- **Local Development**: AWS SAM CLI (Serverless Application Model)
- **Language**: TypeScript
- **Cloud Provider**: AWS Lambda

## Project Structure

```
hono-lambda/
├── bin/
│   └── hono-lambda.ts          # CDK app entry point
├── lib/
│   └── hono-lambda-stack.ts    # CDK stack definition
├── lambda/
│   └── index.ts                # Hono application code
├── test/
│   └── hono-lambda.test.ts     # Unit tests
├── template.yaml               # SAM template for local development
├── .env.local                  # Local environment variables
├── package.json                # Dependencies and scripts
├── tsconfig.json              # TypeScript configuration
└── cdk.json                   # CDK configuration
```

## Setup Instructions

### Prerequisites

1. **Install dependencies**:
   ```bash
   npm install
   npm install -g aws-cdk aws-sam-cli
   ```

2. **Install dotenv-cli** for environment management:
   ```bash
   npm install --save-dev dotenv-cli
   ```

### Environment Configuration

Create `.env.local` for local development:
```bash
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
AWS_DEFAULT_REGION=us-east-1
AWS_SESSION_TOKEN=test
NODE_ENV=development
AWS_PAGER=
```

## Development Workflow

### Available Scripts

```json
{
  "build": "tsc",
  "watch": "tsc -w",
  "dev": "npm run build && dotenv -e .env.local sam local start-api",
  "dev:watch": "npm run build && dotenv -e .env.local sam local start-api --warm-containers EAGER",
  "dev:debug": "npm run build && dotenv -e .env.local sam local start-api --debug-port 5858",
  "sam:build": "sam build",
  "sam:validate": "sam validate",
  "sam:invoke": "dotenv -e .env.local sam local invoke HonoFunction",
  "deploy:dev": "cdk deploy --context environment=dev",
  "deploy:prod": "cdk deploy --context environment=prod"
}
```

### Local Development

1. **Start development server**:
   ```bash
   npm run dev
   ```
   Access at: http://127.0.0.1:3000

2. **Watch mode** (auto-reload):
   ```bash
   npm run dev:watch
   ```

3. **Debug mode**:
   ```bash
   npm run dev:debug
   ```

### Testing

- **Direct Lambda invoke**:
  ```bash
  npm run sam:invoke
  ```

- **Validate SAM template**:
  ```bash
  npm run sam:validate
  ```

## Deployment

### Development Environment
```bash
npm run deploy:dev
```

### Production Environment
```bash
npm run deploy:prod
```

## Key Configuration Files

### 1. TypeScript Configuration (`tsconfig.json`)

Critical fix for Hono compatibility:
```json
{
  "compilerOptions": {
    "lib": ["es2022", "dom"],  // "dom" required for Web API types
    "target": "ES2022",
    "module": "CommonJS"
  }
}
```

### 2. SAM Template (`template.yaml`)

Local development configuration:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  HonoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .aws-sam/build/HonoFunction/
      Handler: index.handler
      Runtime: nodejs22.x
      Events:
        Api:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY
        Root:
          Type: Api
          Properties:
            Path: /
            Method: ANY
```

### 3. CDK Stack (`lib/hono-lambda-stack.ts`)

Production deployment configuration:
```typescript
import * as cdk from 'aws-cdk-lib'
import { Construct } from 'constructs'
import * as lambda from 'aws-cdk-lib/aws-lambda'
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs'

export class HonoLambdaStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const fn = new NodejsFunction(this, 'lambda', {
      entry: 'lambda/index.ts',
      handler: 'handler',
      runtime: lambda.Runtime.NODEJS_22_X,
    })
    
    const fnUrl = fn.addFunctionUrl({
      authType: lambda.FunctionUrlAuthType.NONE,
    })
    
    new cdk.CfnOutput(this, 'lambdaUrl', {
      value: fnUrl.url!
    })
  }
}
```

### 4. Hono Application (`lambda/index.ts`)

```typescript
import { Hono } from 'hono'
import { handle } from 'hono/aws-lambda'

const app = new Hono()

app.get('/', (c) => c.text('Hello Hono!'))

export const handler = handle(app)
```

## Architecture Decisions

### Local vs Production Environment

| Aspect | Local (SAM) | Production (CDK) |
|--------|-------------|------------------|
| **API Gateway** | ✅ Full API Gateway | ❌ Direct Function URL |
| **Debugging** | ✅ Easy debugging | ⚠️ CloudWatch logs |
| **Testing** | ✅ Request/response inspection | ⚠️ Limited visibility |
| **Performance** | ⚠️ Slower (more layers) | ✅ Faster (direct invoke) |
| **Cost** | ✅ Free locally | ✅ Cheaper (no API Gateway) |

### Why This Mixed Approach?

1. **AWS Official Recommendation**: SAM CLI + CDK is the recommended pattern
2. **Development Benefits**: API Gateway provides better debugging and testing capabilities
3. **Production Efficiency**: Function URL is simpler and more cost-effective
4. **Best of Both Worlds**: Comprehensive local testing with efficient production deployment

## Troubleshooting

### Common Issues

1. **TypeScript compilation error**: "Cannot find name 'RequestInfo'"
   - **Solution**: Add "dom" to lib array in `tsconfig.json`

2. **AWS credentials error**: "Token has expired"
   - **Solution**: Use `.env.local` with test credentials for local development
   - SAM CLI doesn't need real AWS credentials for local testing

3. **SAM validation errors**
   - **Solution**: Run `npm run sam:validate` to check template syntax

4. **Build errors**
   - **Solution**: Ensure TypeScript compilation succeeds before SAM operations

### Performance Notes

- **Cold Start**: ~22ms initialization, ~1.4s response time for local Lambda
- **Warm Containers**: Use `npm run dev:watch` for faster subsequent requests
- **Memory Usage**: ~128MB typical usage

## Key Learnings

1. **AWS Architecture**: SAM CLI and CDK are complementary, not competitive
2. **Environment Parity**: Local environment closely matches Lambda runtime using Docker
3. **Development Experience**: API Gateway provides superior debugging vs direct Function URL
4. **Cost Optimization**: Function URL reduces costs by eliminating API Gateway charges
5. **TypeScript Integration**: Proper lib configuration essential for web framework compatibility

## Next Steps

- Add more endpoints to the Hono application
- Implement environment-specific configurations
- Add comprehensive testing suite
- Set up CI/CD pipeline
- Add monitoring and logging
- Implement authentication/authorization

## References

- [Hono Documentation](https://hono.dev/)
- [AWS SAM CLI Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/)
- [AWS CDK Guide](https://docs.aws.amazon.com/cdk/)
- [AWS Lambda Function URLs](https://docs.aws.amazon.com/lambda/latest/dg/lambda-urls.html) 
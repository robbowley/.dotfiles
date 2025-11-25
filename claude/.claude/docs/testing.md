# Testing Principles

## Behavior-Driven Testing

- **No "unit tests"** - this term is not helpful. Tests should verify expected behavior, treating implementation as a black box
- Test through the public API exclusively - internals should be invisible to tests
- No 1:1 mapping between test files and implementation files
- Tests that examine internal implementation details are wasteful and should be avoided
- **Coverage targets**: 100% coverage should be expected at all times, but these tests must ALWAYS be based on business behaviour, not implementation details
- Tests must document expected business behaviour

## Testing Tools

- **Jest** or **Vitest** for testing frameworks
- **React Testing Library** for React components
- **MSW (Mock Service Worker)** for API mocking when needed
- All test code must follow the same TypeScript strict mode rules as production code

## Vitest Process Cleanup

**Context**: When running tests with Vitest in projects

**Issue**: Vitest spawns worker processes (`vitest/dist/workers/forks.js`) that can become orphaned if not cleaned up properly. These processes:
- Consume 20-30% CPU each
- Use significant memory (200-500MB per worker)
- Persist across Claude Code sessions
- Can accumulate and make the system unusable

**Solution**: Always ensure vitest processes are cleaned up after running tests.

**Required setup in projects using Vitest:**

Add cleanup scripts to `package.json`:
```json
{
  "scripts": {
    "test:cleanup": "pkill -f 'vitest/dist/workers/forks.js' 2>/dev/null || true",
    "test:ci": "npm run test:run && npm run test:cleanup"
  }
}
```

**Test execution pattern:**

```bash
# ✅ CORRECT - Use test:ci which includes cleanup
npm run test:ci

# ✅ CORRECT - Manual cleanup after test:run
npm run test:run
npm run test:cleanup

# ✅ CORRECT - Targeted/single-file tests with cleanup
npm run test:run -- src/features/gap-analyzer.test.ts && npm run test:cleanup
npm run test:run -- --reporter=verbose src/features/my.test.ts && npm run test:cleanup

# ❌ WRONG - Leaves orphaned processes
npm test  # This starts watch mode
npm run test:run -- src/features/my.test.ts  # No cleanup!
```

**Implementation guidelines:**
- **Always use `npm run test:ci`** instead of `npm test` or `npm run test:run`
- If a project doesn't have `test:ci` script, add it first
- After any failed test run, manually run cleanup: `npm run test:cleanup`
- Never use `npm test` as it often starts watch mode

**Why this matters:**
- Multiple orphaned workers make user's machine unusable
- User has to manually kill processes if not cleaned up
- Creates frustration and breaks trust in Claude Code
- Easy to prevent with proper cleanup

## Test Organization

```
src/
  features/
    payment/
      payment-processor.ts
      payment-validator.ts
      payment-processor.test.ts // The validator is an implementation detail. Validation is fully covered, but by testing the expected business behaviour, treating the validation code itself as an implementation detail
```

## Test Data Pattern

Use factory functions with optional overrides for test data:

```typescript
const getMockPaymentPostPaymentRequest = (
  overrides?: Partial<PostPaymentsRequestV3>
): PostPaymentsRequestV3 => {
  return {
    CardAccountId: "1234567890123456",
    Amount: 100,
    Source: "Web",
    AccountStatus: "Normal",
    LastName: "Doe",
    DateOfBirth: "1980-01-01",
    PayingCardDetails: {
      Cvv: "123",
      Token: "token",
    },
    AddressDetails: getMockAddressDetails(),
    Brand: "Visa",
    ...overrides,
  };
};

const getMockAddressDetails = (
  overrides?: Partial<AddressDetails>
): AddressDetails => {
  return {
    HouseNumber: "123",
    HouseName: "Test House",
    AddressLine1: "Test Address Line 1",
    AddressLine2: "Test Address Line 2",
    City: "Test City",
    ...overrides,
  };
};
```

Key principles:

- Always return complete objects with sensible defaults
- Accept optional `Partial<T>` overrides
- Build incrementally - extract nested object factories as needed
- Compose factories for complex objects
- Consider using a test data builder pattern for very complex objects

### When to Extract Test Factories

**Wait for the pattern to emerge.** Don't extract factories prematurely - inline test data is fine initially.

**Extract when you see 5-6+ tests with similar setup patterns:**

```typescript
// ❌ TOO EARLY - Only 2-3 tests, keep inline
it("should categorize matching document", () => {
  const file = {
    relativePath: 'arch.txt',
    fileType: 'txt',
    absolutePath: '/data/arch.txt',
    content: 'System architecture',
  }
  // test continues...
})

// ✅ RIGHT TIME - After 5-6 similar tests, extract factory
const getMockScannedFile = (
  overrides?: Partial<ScannedFile>
): ScannedFile => {
  return {
    relativePath: 'architecture/system-design.txt',
    fileType: 'txt',
    absolutePath: '/data-room/architecture/system-design.txt',
    content: 'System architecture using microservices',
    ...overrides,
  }
}

it("should categorize matching document", () => {
  const file = getMockScannedFile()
  // test continues...
})

it("should handle different file types", () => {
  const file = getMockScannedFile({ fileType: 'pdf' })
  // test continues...
})
```

**Benefits of waiting:**
- Pattern becomes clear after multiple uses
- Avoid premature abstraction
- Factory signature evolves from real needs
- Easier to identify which fields commonly vary (use as overrides)

## Validating Test Data

When schemas exist, validate factory output to catch test data issues early:

```typescript
import { PaymentSchema, type Payment } from '../schemas/payment.schema';

const getMockPayment = (overrides?: Partial<Payment>): Payment => {
  const basePayment = {
    amount: 100,
    currency: "GBP",
    cardId: "card_123",
    customerId: "cust_456",
  };

  const paymentData = { ...basePayment, ...overrides };

  // Validate against real schema to catch type mismatches
  return PaymentSchema.parse(paymentData);
};

// This catches errors in test setup:
const payment = getMockPayment({
  amount: -100  // ❌ Schema validation fails: amount must be positive
});
```

**Why validate test data:**
- Ensures test factories produce valid data that matches production schemas
- Catches test data bugs immediately rather than in test assertions
- Documents constraints (e.g., "amount must be positive") in schema, not in every test
- Prevents tests from passing with invalid data that would fail in production

## Anti-Patterns in Tests

**Avoid these test smells:**

```typescript
// ❌ BAD - Implementation-focused test
it("should call validateAmount", () => {
  const spy = jest.spyOn(validator, 'validateAmount');
  processPayment(payment);
  expect(spy).toHaveBeenCalled();
});

// ✅ GOOD - Behavior-focused test
it("should reject payments with negative amounts", () => {
  const payment = getMockPayment({ amount: -100 });
  const result = processPayment(payment);
  expect(result.success).toBe(false);
  expect(result.error.message).toBe("Invalid amount");
});

// ❌ BAD - Using let and beforeEach (shared mutable state)
let payment: Payment;
beforeEach(() => {
  payment = { amount: 100 };
});
it("should process payment", () => {
  processPayment(payment);
});

// ✅ GOOD - Factory functions (isolated, immutable)
it("should process payment", () => {
  const payment = getMockPayment({ amount: 100 });
  processPayment(payment);
});
```

## Achieving 100% Coverage Through Business Behavior

Example showing how validation code gets 100% coverage without testing it directly:

```typescript
// payment-validator.ts (implementation detail)
export const validatePaymentAmount = (amount: number): boolean => {
  return amount > 0 && amount <= 10000;
};

export const validateCardDetails = (card: PayingCardDetails): boolean => {
  return /^\d{3,4}$/.test(card.cvv) && card.token.length > 0;
};

// payment-processor.ts (public API)
export const processPayment = (
  request: PaymentRequest
): Result<Payment, PaymentError> => {
  // Validation is used internally but not exposed
  if (!validatePaymentAmount(request.amount)) {
    return { success: false, error: new PaymentError("Invalid amount") };
  }

  if (!validateCardDetails(request.payingCardDetails)) {
    return { success: false, error: new PaymentError("Invalid card details") };
  }

  // Process payment...
  return { success: true, data: executedPayment };
};

// payment-processor.test.ts
describe("Payment processing", () => {
  // These tests achieve 100% coverage of validation code
  // without directly testing the validator functions

  it("should reject payments with negative amounts", () => {
    const payment = getMockPaymentPostPaymentRequest({ amount: -100 });
    const result = processPayment(payment);

    expect(result.success).toBe(false);
    expect(result.error.message).toBe("Invalid amount");
  });

  it("should reject payments exceeding maximum amount", () => {
    const payment = getMockPaymentPostPaymentRequest({ amount: 10001 });
    const result = processPayment(payment);

    expect(result.success).toBe(false);
    expect(result.error.message).toBe("Invalid amount");
  });

  it("should reject payments with invalid CVV format", () => {
    const payment = getMockPaymentPostPaymentRequest({
      payingCardDetails: { cvv: "12", token: "valid-token" },
    });
    const result = processPayment(payment);

    expect(result.success).toBe(false);
    expect(result.error.message).toBe("Invalid card details");
  });

  it("should process valid payments successfully", () => {
    const payment = getMockPaymentPostPaymentRequest({
      amount: 100,
      payingCardDetails: { cvv: "123", token: "valid-token" },
    });
    const result = processPayment(payment);

    expect(result.success).toBe(true);
    expect(result.data.status).toBe("completed");
  });
});
```

## React Component Testing

```typescript
// Good - testing user-visible behavior
describe("PaymentForm", () => {
  it("should show error when submitting invalid amount", async () => {
    render(<PaymentForm />);

    const amountInput = screen.getByLabelText("Amount");
    const submitButton = screen.getByRole("button", { name: "Submit Payment" });

    await userEvent.type(amountInput, "-100");
    await userEvent.click(submitButton);

    expect(screen.getByText("Amount must be positive")).toBeInTheDocument();
  });
});
```

## Outside-In TDD with Mocked Dependencies

**Context**: When building features that depend on external systems or complex integrations (APIs, LLMs, databases)

**Pattern**: Define the interface first, mock it in tests, implement the real integration separately. This is "outside-in" development.

**Benefits:**
- Fast test execution (no network calls, no expensive operations)
- Clear contracts and interfaces
- Decoupled architecture
- Can develop feature logic before integration exists

**Example:**

```typescript
// Step 1: Define the interface for the dependency
export type LlmService = {
  categorize: (params: { content: string; description: string }) => LlmCategorizationResult
}

export type LlmCategorizationResult = {
  matched: boolean
  confidence: number
  reasoning: string
}

// Step 2: Write tests with mocked dependency (using vitest)
import { vi } from 'vitest'

const getMockLlmService = (
  overrides?: Partial<LlmCategorizationResult>
): LlmService => {
  const defaultResult: LlmCategorizationResult = {
    matched: false,
    confidence: 0.5,
    reasoning: 'Mock response',
    ...overrides,
  }

  return {
    categorize: vi.fn().mockReturnValue(defaultResult),
  }
}

it('should return matched true when LLM determines content matches', () => {
  const mockLlmService = getMockLlmService({
    matched: true,
    confidence: 0.95,
    reasoning: 'Document contains system architecture information',
  })

  const result = categorizeDocument({
    file: scannedFile,
    requestedItem: requestedItem,
    llmService: mockLlmService,  // Inject the mock
  })

  expect(result.matched).toBe(true)
  expect(mockLlmService.categorize).toHaveBeenCalledWith({
    content: 'System architecture using microservices',
    description: 'System architecture diagrams and documentation',
  })
})

// Step 3: Implement feature using dependency injection
type CategorizeDocumentOptions = {
  file: ScannedFile
  requestedItem: RequestedInformationItem
  llmService: LlmService  // Accept interface, not concrete implementation
}

export const categorizeDocument = (
  options: CategorizeDocumentOptions
): CategorizationResult => {
  const { file, requestedItem, llmService } = options

  const llmResult = llmService.categorize({
    content: file.content || '',
    description: requestedItem.description,
  })

  return {
    matched: llmResult.matched,
  }
}

// Step 4: Later, implement the real integration
export const createClaudeLlmService = (apiKey: string): LlmService => {
  return {
    categorize: (params) => {
      // Real Claude API call here
      const response = callClaudeAPI(params.content, params.description)
      return {
        matched: response.matched,
        confidence: response.confidence,
        reasoning: response.reasoning,
      }
    },
  }
}

// Step 5: Wire up in production
const llmService = createClaudeLlmService(process.env.CLAUDE_API_KEY)
const result = categorizeDocument({
  file: myFile,
  requestedItem: myItem,
  llmService: llmService,  // Real implementation
})
```

**Key principles:**
- Define clear interface boundaries (use TypeScript `type` for data, `interface` for behavior contracts)
- Mock at the integration boundary, not internal functions
- Tests should verify your feature logic, not the external system
- Real integration implemented and tested separately

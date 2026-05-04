# Unit Testing Rules

## Overview

All new features and bug fixes in the Aya AI project must include appropriate unit tests. Tests prevent regressions, document expected behavior, and ensure code quality.

## Test Categories

### 1. Unit Tests

- Test individual functions/methods in isolation
- Fast execution, no external dependencies
- Mock external services (S3, Replicate, AWS, etc.)

### 2. Integration Tests

- Test multiple components working together
- May use real database (test instance)
- Example: `test_endpoints.py` (live API tests)

### 3. End-to-End Tests

- Full user flows across frontend/backend
- Browser automation (Playwright/Cypress)
- Not covered in this document

**Important Note**: `backend/test_endpoints.py` is an integration test that hits live URLs, NOT a unit test. True unit tests should never make external network calls.

## When Tests Are Required

### Always Required

- **Bug fixes** - Must include regression test
- **Business logic** - Points, licenses, payments, permissions
- **Utility functions** - Date calculations, transformations, validations
- **Data models** - Custom methods, properties, validations
- **External API integrations** - Replicate, S3, ComfyUI workflows

### Required Based on Complexity

- **New API endpoints** - Required if >3 parameters or business logic
- **Database operations** - Required for non-trivial queries/updates
- **Error handling** - Required for custom error paths
- **Async operations** - Required for complex flows

### Optional (But Encouraged)

- Simple CRUD operations (basic GET/POST/PUT/DELETE)
- UI components without state/logic
- Configuration/constant definitions

## Backend Testing (FastAPI)

### Test Structure
```
backend/tests/
├── unit/
│   ├── test_auth.py
│   ├── test_license_utils.py
│   └── test_<feature>.py
├── integration/
│   ├── test_endpoints.py
│   └── test_workflows.py
└── conftest.py
```

### Testing Patterns

#### 1. Pure Function Tests
```python
# Example: test_license_utils.py
class TestCalculateLicenseExpiration:
    def test_perpetual_license(self):
        """Test that perpetual licenses return None."""
        created_at = datetime(2024, 1, 1, 12, 0, 0, tzinfo=timezone.utc)
        result = calculate_license_expiration(created_at, 1, "perpetual")
        assert result is None
    
    def test_edge_cases(self):
        """Test edge cases and error conditions."""
        with pytest.raises(ValueError, match="Unknown duration"):
            calculate_license_expiration(now, 1, "invalid")
```

#### 2. API Endpoint Tests
```python
# Example: test_face_swap_workflow.py
@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def mock_user():
    user = Mock(spec=User)
    user.id = TEST_USER_ID
    return user

@pytest.fixture(autouse=True)
def override_dependencies(mock_user, mock_db):
    app.dependency_overrides[get_current_user] = lambda: mock_user
    app.dependency_overrides[get_db] = lambda: mock_db
    yield
    app.dependency_overrides.clear()

@patch('app.routers.feature.external_service')
def test_endpoint_success(mock_service, client, mock_user):
    mock_service.return_value = "mocked_result"
    response = client.post("/api/endpoint", json=data)
    assert response.status_code == 200
    assert response.json()["key"] == "expected_value"
```

#### 3. Database Tests
```python
@pytest.mark.asyncio
async def test_crud_operation():
    async with TestSession() as session:
        # Create test data
        entity = MyModel(name="test")
        session.add(entity)
        await session.commit()
        
        # Test CRUD operation
        result = await my_crud.get(session, entity.id)
        assert result.name == "test"
```

### Must-Test Scenarios

1. **Points System**
   - Point deduction for each service
   - Insufficient balance handling
   - Refund scenarios

2. **License System**
   - Expiration calculations (all time units)
   - Perpetual licenses
   - License validation

3. **File Operations**
   - S3 uploads/downloads
   - Image processing
   - Metadata embedding

4. **Authentication/Authorization**
   - Token validation
   - Permission checks
   - Rate limiting

### Mocking Guidelines

```python
# External APIs
@patch('app.services.replicate.client')
def test_replicate_integration(mock_client):
    mock_client.predictions.create.return_value = Mock(id="pred-123")
    # Test logic

# Database
@pytest.fixture
def mock_db():
    return AsyncMock(spec=AsyncSession)

# S3/AWS
@patch('app.core.s3.client')
def test_s3_upload(mock_s3):
    mock_s3.upload_fileobj.return_value = None
    # Test upload logic
```

### Running Tests

```bash
# All tests
docker compose exec backend pytest

# Specific file
docker compose exec backend pytest tests/unit/test_license_utils.py

# With coverage
docker compose exec backend pytest --cov=app tests/

# Watch mode
docker compose exec backend ptw --runner "python -m pytest"
```

## Frontend Testing (Next.js)

### Setup Required
The frontend currently has no test infrastructure. To add:

1. Install dependencies:

```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom jest jest-environment-jsdom
```

2. Create `jest.config.js`:

```javascript
const nextJest = require('next/jest')
const createJestConfig = nextJest({
  dir: './',
})
const customJestConfig = {
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
}
module.exports = createJestConfig(customJestConfig)
```

### Test Structure
```
frontend/src/
├── __tests__/
│   ├── components/
│   │   ├── ImageGrid.test.tsx
│   │   └── DownloadLicenseModal.test.tsx
│   ├── slices/
│   │   └── generatorSlice.test.ts
│   └── utils/
│       └── api.test.ts
```

### Testing Patterns

#### 1. Component Tests
```typescript
// ImageGrid.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Provider } from 'react-redux'
import { store } from '@/store'
import ImageGrid from '@/components/ImageGrid'

describe('ImageGrid', () => {
  it('renders images correctly', () => {
    render(
      <Provider store={store}>
        <ImageGrid images={mockImages} />
      </Provider>
    )
    
    expect(screen.getAllByRole('img')).toHaveLength(3)
  })
  
  it('handles download click', () => {
    const onDownload = jest.fn()
    render(
      <Provider store={store}>
        <ImageGrid images={mockImages} onDownload={onDownload} />
      </Provider>
    )
    
    fireEvent.click(screen.getByText('Download'))
    expect(onDownload).toHaveBeenCalledWith(mockImages[0].id)
  })
})
```

#### 2. Redux Slice Tests
```typescript
// generatorSlice.test.ts
import { configureStore } from '@reduxjs/toolkit'
import generatorSlice, { setGenerating, addImage } from '@/store/slices/generatorSlice'

describe('generatorSlice', () => {
  it('should handle setGenerating', () => {
    const state = generatorSlice.reducer(undefined, setGenerating(true))
    expect(state.isGenerating).toBe(true)
  })
  
  it('should add image to gallery', () => {
    const initialState = { images: [] }
    const action = addImage({ id: '123', url: 'test.jpg' })
    const state = generatorSlice.reducer(initialState, action)
    
    expect(state.images).toHaveLength(1)
    expect(state.images[0]).toEqual({ id: '123', url: 'test.jpg' })
  })
})
```

#### 3. API Utility Tests
```typescript
// api.test.ts
import { api } from '@/utils/api'
import MockAdapter from 'axios-mock-adapter'

describe('API Utils', () => {
  let mock: MockAdapter
  
  beforeEach(() => {
    mock = new MockAdapter(api)
  })
  
  it('handles successful response', async () => {
    mock.onGet('/test').reply(200, { data: 'success' })
    
    const response = await api.get('/test')
    expect(response.data).toEqual({ data: 'success' })
  })
  
  it('handles 401 unauthorized', async () => {
    mock.onGet('/test').reply(401)
    
    await expect(api.get('/test')).rejects.toThrow()
  })
})
```

### Must-Test Frontend Scenarios

1. **User Interactions**
   - Form submissions
   - Button clicks
   - File uploads

2. **State Management**
   - Redux actions/reducers
   - Local state changes
   - Data fetching

3. **Error Handling**
   - API errors
   - Validation messages
   - Network failures

4. **Navigation**
   - Route changes
   - Protected routes
   - Query parameters

## Admin Panel Testing (Next.js)

Follows the same patterns as frontend testing with additional focus on:

### Admin-Specific Tests

1. **CRUD Operations**

```typescript
// TalentManagement.test.tsx
describe('TalentManagement', () => {
  it('creates new talent', async () => {
    const mockCreate = jest.fn()
    render(<TalentManagement onCreate={mockCreate} />)
    
    fireEvent.change(screen.getByLabelText('Name'), { target: { value: 'Test Talent' } })
    fireEvent.click(screen.getByText('Create'))
    
    await waitFor(() => {
      expect(mockCreate).toHaveBeenCalledWith({ name: 'Test Talent' })
    })
  })
})
```

2. **Data Tables**

```typescript
// DataTable.test.tsx
describe('DataTable', () => {
  it('sorts columns correctly', () => {
    render(<DataTable data={mockData} columns={columns} />)
    
    fireEvent.click(screen.getByText('Name'))
    expect(screen.getAllByRole('row')[1]).toHaveTextContent('Alice')
  })
})
```

3. **Permissions**

```typescript
// ProtectedRoute.test.tsx
describe('ProtectedRoute', () => {
  it('redirects unauthorized users', () => {
    render(
      <ProtectedRoute requiredRole="admin">
        <div>Admin Content</div>
      </ProtectedRoute>,
      { initialAuthState: { role: 'user' } }
    )
    
    expect(screen.queryByText('Admin Content')).not.toBeInTheDocument()
  })
})
```

## Bug Fix Testing Requirements

### Regression Test Template

```python
# test_<bug_name>_fix.py
def test_<bug_description>_regression():
    """
    Regression test for: <brief bug description>
    GitHub Issue: #<issue_number>
    """
    # Setup scenario that previously caused bug
    # Trigger the bug condition
    # Verify fix works correctly
    assert expected_behavior
```

### Example

```python
def test_license_expiration_timezone_handling_regression():
    """
    Regression test for: License expiration incorrect for timezone-naive dates
    GitHub Issue: #1234
    """
    created_at = datetime(2024, 1, 1, 12, 0, 0)  # No timezone
    result = calculate_license_expiration(created_at, 1, "day")
    expected = datetime(2024, 1, 2, 12, 0, 0, tzinfo=timezone.utc)
    assert result == expected
```

## Test Quality Standards

### Good Tests

1. **Clear naming** - `test_scenario_expected_result`
2. **Single responsibility** - One behavior per test
3. **AAA pattern** - Arrange, Act, Assert
4. **Descriptive assertions** - Custom error messages
5. **Edge case coverage** - Null, empty, invalid inputs

### Bad Tests to Avoid

1. **Testing implementation** - Focus on behavior, not internals
2. **Flaky tests** - No delays, no race conditions
3. **Over-mocking** - Mock only external dependencies
4. **Complex setup** - Keep fixtures simple
5. **No assertions** - Every test must verify something

## Coverage Requirements

- **Critical paths**: 100% coverage (points, licenses, auth)
- **Business logic**: 90%+ coverage
- **Utilities**: 95%+ coverage
- **UI components**: 50%+ coverage (when implemented) *Note: This is a future goal as frontend test infrastructure needs to be established*
- **Overall**: Maintain 80%+ coverage

## CI/CD Integration

Tests run automatically on:

1. **Pull requests** - All tests must pass
2. **Main branch** - Full test suite with coverage
3. **Nightly** - Integration tests

## Common Test Fixtures (`conftest.py`)

Create `backend/tests/conftest.py` for shared fixtures:

```python
import pytest
from unittest.mock import Mock, AsyncMock
from fastapi.testclient import TestClient
from sqlalchemy.ext.asyncio import AsyncSession
from app.main import app
from app.models.user import User

@pytest.fixture
def client():
    """Create test client."""
    return TestClient(app)

@pytest.fixture
def mock_user():
    """Create mock authenticated user."""
    user = Mock(spec=User)
    user.id = "123e4567-e89b-12d3-a456-426614174000"
    user.email = "test@example.com"
    user.is_active = True
    return user

@pytest.fixture
def mock_db():
    """Create mock database session."""
    return AsyncMock(spec=AsyncSession)

@pytest.fixture(autouse=True)
def override_dependencies(mock_user, mock_db):
    """Override auth and DB dependencies for all tests."""
    from app.core.security import get_current_user
    from app.db.session import get_db
    
    app.dependency_overrides[get_current_user] = lambda: mock_user
    app.dependency_overrides[get_db] = lambda: mock_db
    yield
    app.dependency_overrides.clear()

@pytest.fixture
def test_image_base64():
    """Create a test 1x1 PNG image in base64."""
    return "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8/5+hHgAHggJ/PchI7wAAAABJRU5ErkJggg=="
```

## Documentation

1. Document complex test scenarios in code comments
2. Update README.md in test directories
3. Include setup instructions for new developers
4. Document any special test data requirements

## Review Checklist

Before submitting code, verify:

- [ ] New features have appropriate tests
- [ ] Bug fixes include regression tests
- [ ] All tests pass locally
- [ ] Coverage meets requirements
- [ ] No flaky tests
- [ ] Test names are descriptive
- [ ] Tests are independent (no shared state)

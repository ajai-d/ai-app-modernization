### **1. Initialization**
1. **Input Gathering**:
   - Load the PRD, old codebase, and commit history.
   - Parse any existing documentation or architectural notes.
   - Load test plans and any existing test results.

2. **Environment Setup**:
   - Set up analysis tools for codebase parsing and documentation generation.
   - Configure observability tools for later runtime monitoring.

---

### **2. Intermediate Spec Generation**
#### **Step 2.0: Iterative Design Process**
   - **Phase 2.0.1: High-Level Design**
     - **Domain Analysis**:
       - Extract business domains and bounded contexts from PRD
       - Identify core business entities and their relationships
       - Map business processes to domain services
       - Define domain boundaries and integration points
     
     - **Application Architecture Design**:
       - Define overall application architecture pattern (layered, hexagonal, microservices, etc.)
       - Identify major application components and their responsibilities
       - Design inter-component communication patterns
       - Define cross-cutting concerns (logging, security, caching, etc.)
     
     - **Project Structure Design**:
       - Define solution structure and project organization
       - Identify shared libraries and common components
       - Plan dependency management and versioning strategy
       - Design build and deployment structure
     
     - **Technology Stack Selection**:
       - Choose frameworks based on requirements and constraints
       - Select third-party packages and libraries
       - Define database and data storage solutions
       - Choose deployment and hosting technologies
     
     - **Integration Architecture**:
       - Design external system integrations
       - Define API contracts and communication protocols
       - Plan authentication and authorization strategies
       - Design data synchronization and event handling

   - **Phase 2.0.2: Mid-Level Design**
     - **Service Layer Design**:
       - Define application services and their interfaces
       - Design service orchestration and coordination
       - Plan transaction boundaries and consistency models
       - Define service contracts and data transfer objects
     
     - **Data Access Design**:
       - Design repository patterns and data access layers
       - Define entity models and relationships
       - Plan data mapping and transformation strategies
       - Design caching and performance optimization
     
     - **API Design**:
       - Define REST/GraphQL API structures
       - Design request/response models
       - Plan API versioning and backward compatibility
       - Define error handling and status codes
     
     - **Security Design**:
       - Design authentication and authorization flows
       - Define security policies and access controls
       - Plan data encryption and protection strategies
       - Design audit logging and compliance measures

   - **Phase 2.0.3: Low-Level Design**
     - **Class Design**:
       - Define domain entities and value objects
       - Design aggregate roots and domain services
       - Plan class hierarchies and inheritance structures
       - Define interfaces and abstractions
     
     - **Method Design**:
       - Define method signatures and contracts
       - Plan parameter validation and error handling
       - Design method composition and reusability
       - Define async/await patterns and concurrency handling
     
     - **Design Pattern Application**:
       - Apply appropriate Gang of Four patterns (Factory, Strategy, Observer, etc.)
       - Implement domain-driven design patterns (Repository, Specification, etc.)
       - Use architectural patterns (CQRS, Event Sourcing, etc.)
       - Apply SOLID principles and clean code practices
     
     - **Error Handling Design**:
       - Define exception hierarchies and custom exceptions
       - Design error propagation and handling strategies
       - Plan logging and diagnostic information capture
       - Define user-friendly error messages and codes

   - **Phase 2.0.4: Design Validation and Refinement**
     - **Design Review Process**:
       - Validate design against PRD requirements
       - Check compliance with architectural principles
       - Review for performance and scalability concerns
       - Assess maintainability and testability
     
     - **Best Practices Validation**:
       - Apply SOLID principles review
       - Check design patterns usage appropriateness
       - Validate separation of concerns
       - Review dependency injection and inversion of control
     
     - **Iterative Refinement**:
       - Iterate on design based on validation feedback
       - Refine component boundaries and responsibilities
       - Optimize for performance and resource usage
       - Simplify complex designs and reduce coupling
     
     - **Design Documentation**:
       - Create detailed design specifications
       - Document design decisions and rationale
       - Generate class diagrams and interaction diagrams
       - Create implementation guidelines and standards

   - **Output**: Comprehensive design specifications including:
     - High-level architecture documentation
     - Component and service designs
     - Class and interface definitions
     - Design pattern implementations
     - Technology stack justifications
     - Integration and deployment designs

#### **Step 2.1: Architecture Diagram Generation**
   - Analyze the old codebase to extract system components, dependencies, and data flow.
   - Generate a new system architecture that modernizes patterns and technologies.
   - **Output**: Multi-format visual diagrams + natural language descriptions of:
     - **Mermaid Diagrams** (Primary format - GitHub-native, AI-friendly):
       - System overview flowchart
       - Component interaction diagrams
       - Data flow diagrams
       - Deployment architecture
     - **Natural Language Descriptions**:
       - System components and their responsibilities
       - Data flow between components
       - External integrations and dependencies
       - Technology stack choices and rationale
     - **Diagram Generation Guidelines**:
       - Use Mermaid syntax for easy GitHub rendering and version control
       - Follow C4 model principles for clear architectural hierarchy
       - Include both high-level overview and detailed component diagrams
       - Generate ASCII art fallbacks for text-only environments
       - Provide diagram source code alongside rendered images

#### **Step 2.2: Functional Spec Generation**
   - Map PRD requirements to specific features and workflows.
   - Analyze existing code to understand current feature implementations.
   - **Output**: Detailed natural language descriptions including:
     - Individual feature descriptions with user stories
     - Step-by-step workflow descriptions
     - Business logic and rules
     - Edge cases and error scenarios
     - Input/output specifications

#### **Step 2.3: UI/UX Spec Generation**
   - Analyze existing UI patterns and user interactions.
   - Map PRD requirements to UI components and user flows.
   - **Output**: Natural language descriptions of:
     - Page layouts and component hierarchies
     - User interaction patterns and behaviors
     - Navigation flows and state transitions
     - Responsive design considerations
     - Accessibility requirements

#### **Step 2.4: Technical Spec Generation**
   - Analyze current technology stack and identify modernization opportunities.
   - Define new technical architecture based on modern patterns.
   - **Output**: Detailed descriptions of:
     - Technology stack choices (frameworks, libraries, databases)
     - API design patterns and standards
     - Data models and schemas
     - Security and authentication patterns
     - Performance and scalability considerations
     - Testing strategy and approaches

#### **Step 2.5: Style Guide Generation**
   - Extract existing coding patterns and conventions from the old codebase.
   - Define modern coding standards and best practices.
   - **Output**: Natural language descriptions of:
     - Code organization and file structure
     - Naming conventions for variables, functions, classes
     - Code formatting and style rules
     - Design system tokens and UI standards
     - Documentation standards

#### **Step 2.6: Test Plan Generation**
   - Analyze existing tests and identify coverage gaps.
   - Map features to testing requirements.
   - **Output**: Detailed descriptions of:
     - Unit test requirements for each component
     - Integration test scenarios
     - End-to-end test workflows
     - Performance test criteria
     - Edge case and error condition tests

#### **Step 2.7: Error Handling & Observability Plan Generation**
   - Analyze current error handling patterns.
   - Define modern observability and monitoring approaches.
   - **Output**: Natural language descriptions of:
     - Error handling strategies for each component
     - Logging standards and formats
     - Metrics and monitoring requirements
     - Alerting rules and escalation procedures
     - Debugging and troubleshooting workflows

#### **Step 2.8: API Contract Generation**
   - Extract existing API patterns from the codebase.
   - Generate OpenAPI specifications for new APIs.
   - **Output**: 
     - OpenAPI/Swagger specifications
     - Natural language descriptions of API design principles
     - Request/response schema documentation
     - Authentication and authorization patterns

#### **Step 2.9: Migration Plan Generation**
   - Compare old and new architectures to identify transformation paths.
   - **Output**: Detailed natural language descriptions of:
     - Components to reuse, replace, or deprecate
     - Data migration strategies
     - Deployment and rollout phases
     - Rollback procedures
     - Risk mitigation strategies

#### **Step 2.10: Glossary Generation**
   - Extract domain-specific terms from PRD and codebase.
   - **Output**: Natural language definitions of:
     - Business domain terminology
     - Technical terminology
     - Acronyms and abbreviations
     - System-specific concepts

---

### **3. Spec Validation and Refinement**
1. **Cross-Reference Validation**:
   - Ensure all specs are consistent with each other and the PRD.
   - Identify conflicts or gaps between different specs.

2. **Completeness Check**:
   - Verify that all PRD requirements are addressed in the specs.
   - Ensure all critical system components are documented.

---

### **4. Iterative Code Generation and Function Validation**
1. **Code Generation**:
   - Use the detailed intermediate specs as input for AI-driven code generation.
   - Generate code for each module/feature based on the natural language descriptions.

2. **Initial Validation**:
   - Deploy the generated code and run basic functionality tests.

3. **Iterative Refinement Loop**
   - Compare old and new apps at runtime using automated tools to identify discrepancies
   - Perform root cause analysis to trace discrepancies back to specific specs or code
   - Update the natural language descriptions in the relevant intermediate specs. Refine feature descriptions, workflows, or technical details as needed
   - Performed targeted code regeneration using updated spec descriptions. Focus on specific modules or features that need changes. Changes are small and atomic, with test runs after each change

#### Note on Long-Running Iterations:
Copilot should be instructed to detect long-running iterations and pivot to a new approach with these principles
   - Make smaller, more focused changes
   - Go back to intermediate specs and refine the design for smaller, more focused components to reduce coupling
   - Invoke HITL if iterations are still not productive

---

The key insight here is that the **intermediate specs are living documents** with detailed natural language descriptions that serve as the "source of truth" for code generation. They get refined iteratively as discrepancies are discovered and resolved.
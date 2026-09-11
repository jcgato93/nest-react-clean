# API Controllers

API Controllers are responsible for handling HTTP requests and responses. They act as the entry point for client interactions, delegating business logic to the appropriate use cases in the application layer.

## Defining API Controllers

When defining an API controller, consider the following best practices:
1. **Use Decorators**: Utilize framework-specific decorators (e.g., `@Controller`, `@Get()`, `@Post()`) to define routes and HTTP methods.
2. **Dependency Injection**: Inject use case classes into the controller to delegate business logic.
3. **Request Validation**: Validate incoming requests using DTOs (Data Transfer Objects) to ensure data integrity.
4. **Swagger Documentation**: Use Swagger decorators to document the API endpoints for better maintainability and client understanding.

## Example

```typescript
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { CreateUserDto } from 'src/infrastructure/api/dtos/create-user.dto';
import { UserUseCase } from 'src/application/use-cases/user.use-case';
import { UserMapper } from 'src/application/users/mappers/user.mapper';

@ApiTags('users')
@Controller('users')
export class UserController {
    constructor(private readonly userUseCase: UserUseCase) {}

    @Post()
    @ApiOperation({ summary: 'Crear un nuevo usuario' })
    @ApiResponse({ status: HttpStatus.CREATED, description: 'Usuario creado exitosamente.' })
    @ApiResponse({ status: HttpStatus.BAD_REQUEST, description: 'Solicitud inválida.' })
    async createUser(@Body() createUserDto: CreateUserDto) {
        const newUser = UserMapper.fromCreateDtoToDomain(createUserDto);
        const result = await this.userUseCase.createUser(newUser);
        return UserMapper.fromDomainToResponseDto(result);
    }

    @Get(':id')
    @ApiOperation({ summary: 'Obtener usuario por ID' })
    @ApiResponse({ status: HttpStatus.OK, description: 'Usuario obtenido exitosamente.' })
    @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Usuario no encontrado.' })
    async getUserById(@Param('id') id: string) {
        const result = await this.userUseCase.getUserById(id);
        return UserMapper.fromDomainToResponseDto(result);
    }

    @Post(':companyId/logo')
  @UseGuards(Auth0Guard)
  @UseInterceptors(FileInterceptor('file'))
  @ApiConsumes('multipart/form-data')
  @ApiBody({
    schema: {
      type: 'object',
      properties: {
        file: {
          type: 'string',
          format: 'binary',
        },
      },
    },
  })

  @ApiOperation({ summary: 'Cargar foto de usuario' })
  @ApiBearerAuth()
  @ApiResponse({
    status: HttpStatus.OK,
    description: 'Foto de usuario cargada correctamente.',
    type: UploadPhotoResponseDto,
  })
  @ApiResponse({
    status: HttpStatus.NOT_FOUND,
    description: 'Usuario no encontrado.',
  })
  async uploadPhoto(
    @Param('userId') userId: string,
    @UploadedFile() file: Express.Multer.File,
  ): Promise<UploadPhotoResponseDto> {
    const fileName = await this.uploadUserPhotoUseCaseImpl.execute(
      userId,
      file,
    );

    return UserMapper.toUploadPhotoResponseDto(fileName);
  }
}
```
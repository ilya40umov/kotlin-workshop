# Creating a micro service with Kotlin & Spring Boot

### Requirements

* Accept / return JSON
* Read from / write to MySQL
* Use Redis for caching

### Generating project

* Install [SDKMan](https://sdkman.io/): `curl -s "https://get.sdkman.io" | bash`
* Install spring boot CLI: `sdk install springboot`
* Let's see what we can use: `spring init --list`
* Generate the project `spring init --build gradle --dependencies=web,jdbc,mysql,flyway,data-redis,cache -l kotlin sb-demo`

### Running the dependencies

* Start MySQL & Redis in Docker  `docker-compose up -d`
* To connect to MySQL `mysql -h 127.0.0.1 -u root --password`
* To connect to Redis `redis-cli`

### Configuring & running the app

Rename `application.properties` to `application.yaml` and put the following content in it:
 
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: root
    password: root
  cache:
    type: redis

logging:
  level:
    org.springframework.jdbc.core: trace
```

To run the micro service, use `./gradlew bootRun`.

### FlyWay migrations

Create our first DB migration called `V1__create_characters.sql` under `resources/db/migration`:

```sql
DROP TABLE IF EXISTS characters;

CREATE TABLE characters
(
    id         INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(250) NOT NULL,
    last_name  VARCHAR(250) NOT NULL,
    nick_name  VARCHAR(250) DEFAULT NULL
);

INSERT INTO characters (first_name, last_name, nick_name)
VALUES ('Diego', 'de la Vega', 'Zorro');
```

### Fixing testing dependencies

```kotlin
repositories {
	jcenter()
}
```

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test") {
    exclude(group = "junit", module = "junit")
    exclude(group = "org.mockito", module = "mockito-core")
}
testImplementation("org.junit.jupiter:junit-jupiter-api")
testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
testImplementation("org.amshove.kluent:kluent:1.48") {
    exclude(group = "junit", module = "junit")
    exclude(group = "com.nhaarman.mockitokotlin2", module = "mockito-kotlin")
}
testImplementation("io.mockk:mockk:1.9.3")
testImplementation("com.ninja-squad:springmockk:1.1.2")
```

### Implementation

```kotlin
data class Character(
    val id: Int,
    val firstName: String,
    val lastName: String,
    val nickName: String?
): Serializable
```

```kotlin
object CharacterRowMapper : RowMapper<Character> {
    override fun mapRow(rs: ResultSet, rowNum: Int) = Character(
        id = rs.getInt("id"),
        firstName = rs.getString("first_name"),
        lastName = rs.getString("last_name"),
        nickName = rs.getString("nick_name")
    )
}
```

```kotlin
@Repository
class CharacterRepository(
    private val jdbcTemplate: JdbcTemplate
) {
    @Cacheable("characters")
    fun findById(id: Int): Character? {
        val characters = jdbcTemplate.query("select * from characters where id = ?", arrayOf(id), CharacterRowMapper)
        return when {
            characters.isEmpty() -> null
            characters.size == 1 -> characters[0]
            else -> throw IllegalStateException("Found ${characters.size} characters for #$id")
        }
    }
}
```

```kotlin
@JdbcTest
@ContextConfiguration(classes = [CharacterRepository::class])
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
internal class CharacterRepositoryTest(
    @Autowired val characterRepository: CharacterRepository
) {

    @Test
    fun `'findById' should return character if given id exists`() {
        characterRepository.findById(1)?.nickName shouldEqual "Zorro"
    }
}
```

```kotlin
@RestController
@RequestMapping("/api/character")
class CharacterController(
    private val characterRepository: CharacterRepository
) {
    @GetMapping("/{id}")
    fun getCharacter(
        @PathVariable id: Int
    ): ResponseEntity<Any> {
        val character = characterRepository.findById(id)
        return if (character == null) {
            ResponseEntity(mapOf("message" to "Not found"), HttpStatus.NOT_FOUND)
        } else {
            ResponseEntity(character, HttpStatus.OK)
        }
    }
}
```

```kotlin
@WebMvcTest(CharacterController::class)
internal class CharacterControllerTest(
    @Autowired val mvc: MockMvc
) {
    @MockkBean
    private lateinit var characterRepository: CharacterRepository

    @Test
    fun `'getCharacter' should return 404 when character with provided id can't be found`() {
        every { characterRepository.findById(any()) } returns null

        mvc.perform(get("/api/character/987654321"))
            .andExpect(MockMvcResultMatchers.status().isNotFound)
            .andExpect(MockMvcResultMatchers.content().json("""{"message":"Not found"}"""))
    }
}
```

Run the tests, run the app, then open `http://localhost:8080/api/character/1` in your browser.
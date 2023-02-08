# Developer Assessment – Coding Assignment

## Application Requirements
    • Write a restful api that takes in a name and address. 
    • Persist those inputs.
    • Name field should be considered as a unique set.
    • Return proper messages for validation, duplicates, and errors.
    • Allow updates to those inputs.
    • Allow deletes to those inputs.
    • Return all inputs persisted.

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace RestfulAPI.Models
{
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Address { get; set; }
    }

    public class UserContext : DbContext
    {
        public UserContext(DbContextOptions<UserContext> options) : base(options)
        {
        }

        public DbSet<User> Users { get; set; }
    }

    [Route("api/[controller]")]
    [ApiController]
    public class UserController : ControllerBase
    {
        private readonly UserContext _context;

        public UserController(UserContext context)
        {
            _context = context;
        }

        [HttpGet]
        public async Task<ActionResult<IEnumerable<User>>> GetUsers()
        {
            return await _context.Users.ToListAsync();
        }

        [HttpPost]
        public async Task<ActionResult<User>> PostUser(User user)
        {
            var existingUser = await _context.Users.SingleOrDefaultAsync(u => u.Name == user.Name);
            if (existingUser != null)
            {
                return BadRequest("A user with the same name already exists.");
            }

            _context.Users.Add(user);
            await _context.SaveChangesAsync();

            return CreatedAtAction("GetUser", new { id = user.Id }, user);
        }

        [HttpPut("{id}")]
        public async Task<IActionResult> PutUser(int id, User user)
        {
            if (id != user.Id)
            {
                return BadRequest();
            }

            _context.Entry(user).State = EntityState.Modified;
            await _context.SaveChangesAsync();

            return NoContent();
        }

        [HttpDelete("{id}")]
        public async Task<IActionResult> DeleteUser(int id)
        {
            var user = await _context.Users.FindAsync(id);
            if (user == null)
            {
                return NotFound();
            }

            _context.Users.Remove(user);
            await _context.SaveChangesAsync();

            return NoContent();
        }
    }
}
```
## Technical Requirements
    • A .NET Core restful api.
    • Use of Dependency Injection.
    • Make use SOLID principals.
    • Incorporate swagger to test endpoints.
    • The persistence can be an in-memory database or a simple no sql db.
    • Use a caching layer to retrieve persisted data and keep cache current for all updates.
    • Write unit tests making use of mocks.

```
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.OpenApi.Models;
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyAPI
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();

            // Register the in-memory database
            services.AddSingleton<IDatabase>(sp => new InMemoryDatabase());

            // Register the caching layer
            services.AddSingleton<ICache>(sp => new MemoryCache(TimeSpan.FromMinutes(30)));

            // Register the repository, making use of dependency injection
            services.AddTransient<IRepository, Repository>();

            // Add Swagger for API testing
            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
            });
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            // Use Swagger to test endpoints
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
            });

            app.UseRouting();
            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }

    // Implement the IDatabase interface as an in-memory database
    public class InMemoryDatabase : IDatabase
    {
        private readonly List<object> _data = new List<object>();

        public void Add(object item)
        {
            _data.Add(item);
        }

        public IEnumerable<object> GetAll()
        {
            return _data;
        }
    }

    // Implement the ICache interface as an in-memory cache
    public class MemoryCache : ICache
    {
        private readonly TimeSpan _expiration;
        private readonly Dictionary<string, CacheItem> _cache = new Dictionary<string, CacheItem>();

        public MemoryCache(TimeSpan expiration)
        {
            _expiration = expiration;
        }

        public T Get<T>(string key) where T : class
        {
            if (_cache.TryGetValue(key, out var item) && item.Expiration > DateTime.Now)
            {
                return item.Value as T;
            }
            return null;
        }

        public void Set<T>(string key, T value) where T : class
        {
            _cache[key] = new CacheItem
            {
                Expiration = DateTime.Now.Add(_expiration),
                Value = value
            };
        }
    }

    // Implement the IRepository interface, making use of SOLID principles and dependency injection.

  public class Repository : IRepository
    {
    private readonly IDatabase _database;
    private readonly ICache _cache;
    public Repository(IDatabase database, ICache cache)
        {
            _database = database;
            _cache = cache;
        }

        public T Get<T>(string key) where T : class
        {
            var item = _cache.Get<T>(key);
            if (item != null)
            {
                return item;
            }

            item = _database.GetAll().OfType<T>().FirstOrDefault();
            if (item != null)
            {
                _cache.Set(key, item);
            }

            return item;
        }

        public void Add<T>(T item) where T : class
        {
            _database.Add(item);
            _cache.Set(item.GetType().Name, item);
        }
    }

    // Example interface for the database
    public interface IDatabase
    {
        void Add(object item);
        IEnumerable<object> GetAll();
    }

    // Example interface for the cache
    public interface ICache
    {
        T Get<T>(string key) where T : class;
        void Set<T>(string key, T value) where T : class;
    }

    // Example cache item class
    public class CacheItem
    {
        public DateTime Expiration { get; set; }
        public object Value { get; set; }
    }

    // Example repository interface
    public interface IRepository
    {
        T Get<T>(string key) where T : class;
        void Add<T>(T item) where T : class;
    }
}
```

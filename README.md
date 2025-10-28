// Proyecto: KD-Electronics - Persistencia con JDBC
// Estructura de archivos (presentados en un solo documento para revisión):

// File: sql/create_table_productos.sql
/*
CREATE DATABASE IF NOT EXISTS kd_electronics;
USE kd_electronics;

CREATE TABLE IF NOT EXISTS productos (
    id_producto INT AUTO_INCREMENT PRIMARY KEY,
    codigo_producto VARCHAR(50) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    descripcion TEXT,
    precio_base DECIMAL(10,2) NOT NULL CHECK (precio_base >= 0),
    precio_venta DECIMAL(10,2) NOT NULL CHECK (precio_venta >= 0),
    categoria VARCHAR(50),
    cantidad_disponible INT DEFAULT 0 CHECK (cantidad_disponible >= 0),
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
*/

// File: src/main/java/com/kdelectronics/model/Producto.java
package com.kdelectronics.model;

import java.math.BigDecimal;
import java.util.Objects;

public class Producto {
    private Integer idProducto; // null si aun no está en BD
    private String codigoProducto;
    private String nombre;
    private String descripcion;
    private BigDecimal precioBase;
    private BigDecimal precioVenta;
    private String categoria;
    private int cantidadDisponible;

    // Constructor para nuevo producto (sin id)
    public Producto(String codigoProducto, String nombre, String descripcion,
                    BigDecimal precioBase, BigDecimal precioVenta,
                    String categoria, int cantidadDisponible) {
        setCodigoProducto(codigoProducto);
        setNombre(nombre);
        setDescripcion(descripcion);
        setPrecioBase(precioBase);
        setPrecioVenta(precioVenta);
        setCategoria(categoria);
        setCantidadDisponible(cantidadDisponible);
    }

    // Constructor con id (cuando viene de BD)
    public Producto(Integer idProducto, String codigoProducto, String nombre, String descripcion,
                    BigDecimal precioBase, BigDecimal precioVenta,
                    String categoria, int cantidadDisponible) {
        this(codigoProducto, nombre, descripcion, precioBase, precioVenta, categoria, cantidadDisponible);
        this.idProducto = idProducto;
    }

    // Getters y setters con validaciones mínimas
    public Integer getIdProducto() { return idProducto; }
    public void setIdProducto(Integer idProducto) { this.idProducto = idProducto; }

    public String getCodigoProducto() { return codigoProducto; }
    public void setCodigoProducto(String codigoProducto) {
        if (codigoProducto == null || codigoProducto.trim().isEmpty()) {
            throw new IllegalArgumentException("El código del producto no puede estar vacío.");
        }
        this.codigoProducto = codigoProducto.trim();
    }

    public String getNombre() { return nombre; }
    public void setNombre(String nombre) {
        if (nombre == null || nombre.trim().isEmpty()) {
            throw new IllegalArgumentException("El nombre del producto no puede estar vacío.");
        }
        this.nombre = nombre.trim();
    }

    public String getDescripcion() { return descripcion; }
    public void setDescripcion(String descripcion) {
        this.descripcion = descripcion == null ? "" : descripcion.trim();
    }

    public BigDecimal getPrecioBase() { return precioBase; }
    public void setPrecioBase(BigDecimal precioBase) {
        if (precioBase == null || precioBase.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("El precio base debe ser un número >= 0.");
        }
        this.precioBase = precioBase;
    }

    public BigDecimal getPrecioVenta() { return precioVenta; }
    public void setPrecioVenta(BigDecimal precioVenta) {
        if (precioVenta == null || precioVenta.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("El precio de venta debe ser un número >= 0.");
        }
        this.precioVenta = precioVenta;
    }

    public String getCategoria() { return categoria; }
    public void setCategoria(String categoria) {
        this.categoria = categoria == null ? "Sin especificar" : categoria.trim();
    }

    public int getCantidadDisponible() { return cantidadDisponible; }
    public void setCantidadDisponible(int cantidadDisponible) {
        if (cantidadDisponible < 0) {
            throw new IllegalArgumentException("La cantidad disponible no puede ser negativa.");
        }
        this.cantidadDisponible = cantidadDisponible;
    }

    @Override
    public String toString() {
        return "Producto{" +
                "idProducto=" + idProducto +
                ", codigoProducto='" + codigoProducto + '\'' +
                ", nombre='" + nombre + '\'' +
                ", precioBase=" + precioBase +
                ", precioVenta=" + precioVenta +
                ", categoria='" + categoria + '\'' +
                ", cantidadDisponible=" + cantidadDisponible +
                '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Producto)) return false;
        Producto producto = (Producto) o;
        return Objects.equals(codigoProducto, producto.codigoProducto);
    }

    @Override
    public int hashCode() {
        return Objects.hash(codigoProducto);
    }
}


// File: src/main/java/com/kdelectronics/db/ConexionBD.java
package com.kdelectronics.db;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Clase utilitaria para obtener conexiones JDBC. Implementada de forma sencilla
 * usando DriverManager. En proyectos reales, considere usar un DataSource
 * (pool de conexiones) como HikariCP, Apache DBCP, etc.
 */
public class ConexionBD {
    private static final Logger LOGGER = Logger.getLogger(ConexionBD.class.getName());

    // TODO: configurar según su entorno
    private static final String URL = "jdbc:mysql://localhost:3306/kd_electronics?useSSL=false&serverTimezone=UTC";
    private static final String USER = "root";
    private static final String PASSWORD = "1234";

    static {
        try {
            // A partir de JDBC 4.0, el driver se registra automáticamente al estar en el classpath.
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            LOGGER.log(Level.WARNING, "Driver JDBC no encontrado: {0}", e.getMessage());
        }
    }

    public static Connection getConnection() throws SQLException {
        // Abrir una nueva conexión cada vez; el llamador debe cerrarla (try-with-resources)
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }
}


// File: src/main/java/com/kdelectronics/dao/ProductoDAO.java
package com.kdelectronics.dao;

import com.kdelectronics.db.ConexionBD;
import com.kdelectronics.model.Producto;

import java.math.BigDecimal;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * DAO para la entidad Producto. Implementa operaciones CRUD y ejemplos de
 * tratamiento de transacciones. Usa PreparedStatement para evitar inyección SQL
 * y try-with-resources para manejo correcto de recursos.
 */
public class ProductoDAO {
    private static final Logger LOGGER = Logger.getLogger(ProductoDAO.class.getName());

    // Crea la tabla si no existe (útil para pruebas de forma automática)
    public void crearTablaSiNoExiste() {
        String sql = "CREATE TABLE IF NOT EXISTS productos (" +
                "id_producto INT AUTO_INCREMENT PRIMARY KEY, " +
                "codigo_producto VARCHAR(50) UNIQUE NOT NULL, " +
                "nombre VARCHAR(100) NOT NULL, " +
                "descripcion TEXT, " +
                "precio_base DECIMAL(10,2) NOT NULL, " +
                "precio_venta DECIMAL(10,2) NOT NULL, " +
                "categoria VARCHAR(50), " +
                "cantidad_disponible INT DEFAULT 0, " +
                "fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP" +
                ") ENGINE=InnoDB;";

        try (Connection conn = ConexionBD.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
            LOGGER.info("Tabla 'productos' verificada/creada.");
        } catch (SQLException e) {
            LOGGER.log(Level.SEVERE, "Error creando la tabla de productos: {0}", e.getMessage());
        }
    }

    public boolean insertarProducto(Producto p) throws SQLException {
        validarProductoParaInsert(p);
        String sql = "INSERT INTO productos (codigo_producto, nombre, descripcion, precio_base, precio_venta, categoria, cantidad_disponible) " +
                "VALUES (?, ?, ?, ?, ?, ?, ?)";

        try (Connection conn = ConexionBD.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, p.getCodigoProducto());
            ps.setString(2, p.getNombre());
            ps.setString(3, p.getDescripcion());
            ps.setBigDecimal(4, p.getPrecioBase());
            ps.setBigDecimal(5, p.getPrecioVenta());
            ps.setString(6, p.getCategoria());
            ps.setInt(7, p.getCantidadDisponible());

            int filas = ps.executeUpdate();
            if (filas == 0) {
                throw new SQLException("No se insertó ningún registro.");
            }
            try (ResultSet rs = ps.getGeneratedKeys()) {
                if (rs.next()) {
                    p.setIdProducto(rs.getInt(1));
                }
            }
            LOGGER.info("Producto insertado: " + p.getCodigoProducto());
            return true;
        }
    }

    private void validarProductoParaInsert(Producto p) {
        if (p == null) throw new IllegalArgumentException("Producto nulo.");
        // Las validaciones de la entidad ya previenen valores inválidos, pero aquí reforzamos
        if (p.getPrecioVenta().compareTo(p.getPrecioBase()) < 0) {
            LOGGER.warning("El precio de venta es menor al precio base.");
        }
    }

    public boolean actualizarProducto(Producto p) throws SQLException {
        if (p.getIdProducto() == null) throw new IllegalArgumentException("El producto no tiene id (no existe en BD).");
        String sql = "UPDATE productos SET nombre = ?, descripcion = ?, precio_base = ?, precio_venta = ?, categoria = ?, cantidad_disponible = ? WHERE id_producto = ?";

        try (Connection conn = ConexionBD.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, p.getNombre());
            ps.setString(2, p.getDescripcion());
            ps.setBigDecimal(3, p.getPrecioBase());
            ps.setBigDecimal(4, p.getPrecioVenta());
            ps.setString(5, p.getCategoria());
            ps.setInt(6, p.getCantidadDisponible());
            ps.setInt(7, p.getIdProducto());

            int filas = ps.executeUpdate();
            LOGGER.info(() -> "Filas actualizadas: " + filas);
            return filas > 0;
        }
    }

    public boolean eliminarProductoPorId(int id) throws SQLException {
        String sql = "DELETE FROM productos WHERE id_producto = ?";
        try (Connection conn = ConexionBD.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            int filas = ps.executeUpdate();
            LOGGER.info(() -> "Filas eliminadas: " + filas);
            return filas > 0;
        }
    }

    public Producto obtenerPorId(int id) throws SQLException {
        String sql = "SELECT id_producto, codigo_producto, nombre, descripcion, precio_base, precio_venta, categoria, cantidad_disponible FROM productos WHERE id_producto = ?";
        try (Connection conn = ConexionBD.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return mapearProducto(rs);
                }
            }
        }
        return null;
    }

    public List<Producto> listarTodos() throws SQLException {
        List<Producto> lista = new ArrayList<>();
        String sql = "SELECT id_producto, codigo_producto, nombre, descripcion, precio_base, precio_venta, categoria, cantidad_disponible FROM productos";
        try (Connection conn = ConexionBD.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql);
             ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                lista.add(mapearProducto(rs));
            }
        }
        return lista;
    }

    private Producto mapearProducto(ResultSet rs) throws SQLException {
        return new Producto(
                rs.getInt("id_producto"),
                rs.getString("codigo_producto"),
                rs.getString("nombre"),
                rs.getString("descripcion"),
                rs.getBigDecimal("precio_base"),
                rs.getBigDecimal("precio_venta"),
                rs.getString("categoria"),
                rs.getInt("cantidad_disponible")
        );
    }

    // Ejemplo de transacción: insertar varios productos atomically
    public void insertarVariosEnTransaccion(List<Producto> productos) throws SQLException {
        String sql = "INSERT INTO productos (codigo_producto, nombre, descripcion, precio_base, precio_venta, categoria, cantidad_disponible) VALUES (?, ?, ?, ?, ?, ?, ?)";
        try (Connection conn = ConexionBD.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            conn.setAutoCommit(false);
            try {
                for (Producto p : productos) {
                    validarProductoParaInsert(p);
                    ps.setString(1, p.getCodigoProducto());
                    ps.setString(2, p.getNombre());
                    ps.setString(3, p.getDescripcion());
                    ps.setBigDecimal(4, p.getPrecioBase());
                    ps.setBigDecimal(5, p.getPrecioVenta());
                    ps.setString(6, p.getCategoria());
                    ps.setInt(7, p.getCantidadDisponible());
                    ps.executeUpdate();
                    try (ResultSet rs = ps.getGeneratedKeys()) {
                        if (rs.next()) {
                            p.setIdProducto(rs.getInt(1));
                        }
                    }
                }
                conn.commit();
                LOGGER.info("Transacción commit: todos los productos insertados.");
            } catch (SQLException ex) {
                conn.rollback();
                LOGGER.log(Level.SEVERE, "Error en transacción, se realizó rollback: {0}", ex.getMessage());
                throw ex;
            } finally {
                conn.setAutoCommit(true);
            }
        }
    }

    // Ajuste de stock con control de concurrencia a nivel de BD (SELECT FOR UPDATE)
    public boolean ajustarStock(String codigoProducto, int delta) throws SQLException {
        String selectSql = "SELECT id_producto, cantidad_disponible FROM productos WHERE codigo_producto = ? FOR UPDATE";
        String updateSql = "UPDATE productos SET cantidad_disponible = ? WHERE id_producto = ?";

        try (Connection conn = ConexionBD.getConnection()) {
            conn.setAutoCommit(false);
            try (PreparedStatement psSelect = conn.prepareStatement(selectSql)) {
                psSelect.setString(1, codigoProducto);
                try (ResultSet rs = psSelect.executeQuery()) {
                    if (!rs.next()) {
                        conn.rollback();
                        return false; // producto no encontrado
                    }
                    int id = rs.getInt("id_producto");
                    int cantidad = rs.getInt("cantidad_disponible");
                    int nueva = cantidad + delta;
                    if (nueva < 0) {
                        conn.rollback();
                        throw new SQLException("Stock insuficiente para realizar la operación.");
                    }
                    try (PreparedStatement psUpdate = conn.prepareStatement(updateSql)) {
                        psUpdate.setInt(1, nueva);
                        psUpdate.setInt(2, id);
                        psUpdate.executeUpdate();
                    }
                }
                conn.commit();
                return true;
            } catch (SQLException ex) {
                conn.rollback();
                LOGGER.log(Level.SEVERE, "Error ajustando stock: {0}", ex.getMessage());
                throw ex;
            } finally {
                conn.setAutoCommit(true);
            }
        }
    }
}


// File: src/main/java/com/kdelectronics/app/MainApp.java
package com.kdelectronics.app;

import com.kdelectronics.dao.ProductoDAO;
import com.kdelectronics.model.Producto;

import java.math.BigDecimal;
import java.sql.SQLException;
import java.util.Arrays;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Ejemplo de uso del DAO. Esta clase contiene un método main que demuestra
 * inserciones, transacciones, actualización y lectura.
 */
public class MainApp {
    private static final Logger LOGGER = Logger.getLogger(MainApp.class.getName());

    public static void main(String[] args) {
        ProductoDAO dao = new ProductoDAO();
        // Asegurarse de que la tabla existe
        dao.crearTablaSiNoExiste();

        try {
            Producto p1 = new Producto("KD1001", "Auriculares Bluetooth", "Auriculares in-ear con micrófono", new BigDecimal("15.00"), new BigDecimal("25.00"), "Audio", 50);
            Producto p2 = new Producto("KD1002", "Cargador USB-C", "Cargador rápido 30W", new BigDecimal("8.00"), new BigDecimal("12.00"), "Accesorios", 100);

            // Insertar productos individualmente
            dao.insertarProducto(p1);
            dao.insertarProducto(p2);

            // Listar todos
            List<Producto> lista = dao.listarTodos();
            LOGGER.info("Listado de productos:");
            lista.forEach(prod -> LOGGER.info(prod.toString()));

            // Actualizar
            p1.setCantidadDisponible(45); // se vendieron 5
            dao.actualizarProducto(p1);
            LOGGER.info("Producto actualizado: " + dao.obtenerPorId(p1.getIdProducto()));

            // Transacción: insertar varios
            Producto p3 = new Producto("KD1003", "Cable HDMI 2m", "Cable HDMI 2.0", new BigDecimal("4.00"), new BigDecimal("7.99"), "Cable", 200);
            Producto p4 = new Producto("KD1004", "Power Bank 10000mAh", "Batería externa", new BigDecimal("12.00"), new BigDecimal("20.00"), "Batería", 80);
            dao.insertarVariosEnTransaccion(Arrays.asList(p3, p4));

            // Ajuste de stock (ejemplo): venta de 2 unidades de KD1002
            boolean ok = dao.ajustarStock("KD1002", -2);
            LOGGER.info("Ajuste de stock KD1002: " + ok);

            LOGGER.info("Estado final de productos:");
            dao.listarTodos().forEach(prod -> LOGGER.info(prod.toString()));

        } catch (IllegalArgumentException | SQLException ex) {
            LOGGER.log(Level.SEVERE, "Error en operación: {0}", ex.getMessage());
        }
    }
}
